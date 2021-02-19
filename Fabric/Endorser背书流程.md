Endorser背书都干了什么

#### 一、背书流程

​		Peer节点中某些节点会充当背书节点的角色。在SDK发起提案请求之后，背书节点进行响应。
​		背书节点对请求服务的签名提案消息执行启动链码容器、模拟执行提案、签名背书的流程。Endorser背书节点在链码模拟执行结束后调用ESCC背书管理系统链码，利用本地签名者实体对模拟执行结果进行签名背书，然后将模拟执行结果、背书签名、链码执行响应消息等封装为提案响应消息（ProposalResponse类型），并回复给请求客户端。
​		所有客户端提交到账本的普通交易消息都需要经过指定的Endorser背书节点签名认可，并在检查收到足够的签名背书信息之后，才能将签名提案消息、模拟执行结果及其背书信息等打包成签名交易消息（Envelope类型），发送给Orderer节点请求排序出块。
​		背书流程Peer节点需要2个服务：7051的背书服务和7052的链码服务。这2个服务在peer启动的时候均已经注册完成（详见peer的启动流程）
​		注：SDK与Peer节点交互是grpc协议，Peer与Chaincode交互是grpc协议。
​       简单的流程如下图：SDK发起业务请求，在Fabric-sdk封装提案请求，并对请求进行签名，作为grpc的客户端发送背书请求，背书服务监听到背书请求，开始进行处理。首先会对请求报文进行预处理，包括验证提案消息格式与签名的合法性、检查提案消息是否为允许通过外部调用的系统链码、检查交易的唯一性、检查签名提案消息是否满足指定的通道访问权限策略等。接下来会启动链码容器，若没有创建会先进行创建合约容器（所以这就是为什么第一次发起请求会时间较长的原因），创建好容器之后会模拟执行交易，使用链码服务与链码容器进行交互，链码容器是无状态的，所以数据访问状态数据库都是要经过peer进行访问，执行完成之后背书节点会处理数据，将数据分为公有数据和私有数据，公有数据签名打包返回给SDK客户端，私有数据Gossip协议分发给各个peer节点。

![1608695918182](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1608695918182.png)

​		至此背书流程结束。

#### 二、源码分析

​		了解了背书大致流程之后，这部分会介绍背书节点的详细工作（源码分析）

​		背书处理流程的入口函数是(e *Endorser) ProcessProposal(fabric\core\endorser\endorser.go)。该函数是在peer启动时注册在背书服务的处理函数。

​		主要分为4个功能模块：(1)提案消息预处理(2)模拟执行提案(3)处理模拟执行结果(4)对模拟结果签名背书

```go
func (e *Endorser) ProcessProposal(ctx context.Context, signedProp *pb.SignedProposal) (*pb.ProposalResponse, error) {
	...
	endorserLogger.Debug("Entering: request from", addr)

	var chainID string
	var hdrExt *pb.ChaincodeHeaderExtension
	var success bool
	defer func() {
		if hdrExt != nil {
            // 处理错误信息
			...
		}
		endorserLogger.Debug("Exit: request from", addr)
	}()

	// 0 --预处理提案信息，获取提案信息，链相关的信息
	vr, err := e.preProcess(signedProp)
	if err != nil {
		resp := vr.resp
		return resp, err
	}

	prop, hdrExt, chainID, txid := vr.prop, vr.hdrExt, vr.chainID, vr.txid
	// 创建交易模拟器与历史查询执行器
	var txsim ledger.TxSimulator
	var historyQueryExecutor ledger.HistoryQueryExecutor
	if acquireTxSimulator(chainID, vr.hdrExt.ChaincodeId) {
		// 创建交易模拟器对象
		// 创建该交易的交易模拟器
		if txsim, err = e.s.GetTxSimulator(chainID, txid); err != nil {
			return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}, nil
		}

		defer txsim.Done()
	    // 获取历史查询执行器
		if historyQueryExecutor, err = e.s.GetHistoryQueryExecutor(chainID); err != nil {
			return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}, nil
		}
	}

	txParams := &ccprovider.TransactionParams{
		ChannelID:            chainID,
		TxID:                 txid,
		SignedProp:           signedProp,
		Proposal:             prop,
		TXSimulator:          txsim,
		HistoryQueryExecutor: historyQueryExecutor,
	}
	// 1 --模拟执行交易提案
	cd, res, simulationResult, ccevent, err := e.SimulateProposal(txParams, hdrExt.ChaincodeId)
	if err != nil {
		return &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}, nil
	}
	// 处理交易模拟运行结果的响应消息
	if res != nil {
           // 创建背书失败的提案响应消息
			...
        	return pResp, nil
		}
	}

	// 2 -- 处理交易模拟结果
	var pResp *pb.ProposalResponse

	if chainID == "" {
		// 链ID为空，则不需要背书，直接构造提案响应消息返回
		pResp = &pb.ProposalResponse{Response: res}
	} else {
		// 3 -- 对模拟执行结果进行签名
		pResp, err = e.endorseProposal(ctx, chainID, txid, signedProp, prop, res, simulationResult, ccevent, hdrExt.PayloadVisibility, hdrExt.ChaincodeId, txsim, cd)

		// if error, capture endorsement failure metric
		meterLabels := []string{
			"channel", chainID,
			"chaincode", hdrExt.ChaincodeId.Name + ":" + hdrExt.ChaincodeId.Version,
		}

		if err != nil {
			// 处理错误信息
            ...
		}
		if pResp.Response.Status >= shim.ERRORTHRESHOLD {
			// 处理错误信息
			return pResp, nil
		}
	}
	// 将处理结果返回给客户端
	pResp.Response = res

	e.Metrics.SuccessfulProposals.Add(1)
	success = true

	return pResp, nil
}
```

##### 0、数据流梳理

​		由于源码涉及很多的数据结构，现在挑选关键几个流程的数据流进行梳理，方便在看到对应结构体的时候看到所在的位置。

###### 		1）提案请求数据流

​		如下图。背书节点收到的请求结构体是pb.SignedProposal，SDK发起的请求功能函数与参数在标黄部分，实际该部分的功能函数与合约是一致的

![1608189403692](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1608189403692.png)

##### 1、预处理提案信息

preProcess()方法负责预处理签名提案消息，包括验证提案消息格式与签名的合法性、检查提案消息是否为允许通过外部调用的系统链码、检查交易的唯一性、检查签名提案消息是否满足指定的通道访问权限策略等。

```go
// 预处理提案信息
func (e *Endorser) preProcess(signedProp *pb.SignedProposal) (*validateResult, error) {
    // 返回结果
	vr := &validateResult{}
    
	// 1、验证签名提案消息格式与签名的合法性 同时获取提案消息、消息头部及其扩展域
	prop, hdr, hdrExt, err := validation.ValidateProposalMessage(signedProp)

	if err != nil {
		e.Metrics.ProposalValidationFailed.Add(1)
		vr.resp = &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}
		return vr, err
	}

	// 2、获取消息通道头部
	chdr, err := putils.UnmarshalChannelHeader(hdr.ChannelHeader)
	if err != nil {
		vr.resp = &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}
		return vr, err
	}
	// 3、获取消息签名头部
	shdr, err := putils.GetSignatureHeader(hdr.SignatureHeader)
	if err != nil {
		vr.resp = &pb.ProposalResponse{Response: &pb.Response{Status: 500, Message: err.Error()}}
		return vr, err
	}
	// 若是系统链码，检查是否为允许从外部调用的系统链码，cscc lscc qscc
	// 3、检查提案消息头部的hdrExt.ChaincodeId.Name链码名称是否为支持从外部调用的系统链码(CSCC、LSCC与QSCC)
	if e.s.IsSysCCAndNotInvokableExternal(hdrExt.ChaincodeId.Name) {
		...
		return vr, err
	}
	// 4、检查签名提案消息的唯一性
	chainID := chdr.ChannelId
	txid := chdr.TxId
	endorserLogger.Debugf("[%s][%s] processing txid: %s", chainID, shorttxid(txid), txid)

	if chainID != "" {
		// 校验txid，确保没有重复
		meterLabels := []string{
			"channel", chainID,
			"chaincode", hdrExt.ChaincodeId.Name + ":" + hdrExt.ChaincodeId.Version,
		}
		// err == nil 查询成功，说明链上已有这个交易，重复执行则报错
		if _, err = e.s.GetTransactionByID(chainID, txid); err == nil {
			...
			return vr, err
		}

		// 检查签名提案消息是否是满足指定通道的访问权限策略，以确保允许交易
		// 确保是用户链码
		if !e.s.IsSysCC(hdrExt.ChaincodeId.Name) {
			// 检查提案是否符合WRITER写通道权限策略
			if err = e.s.CheckACL(signedProp, chdr, shdr, hdrExt); err != nil {
				...
				return vr, err
			}
		}
	} else { 
	}
	vr.prop, vr.hdrExt, vr.chainID, vr.txid = prop, hdrExt, chainID, txid
	return vr, nil
}
```

##### 2、模拟执行交易

```go
// 模拟执行提案
func (e *Endorser) SimulateProposal(txParams *ccprovider.TransactionParams, cid *pb.ChaincodeID) (ccprovider.ChaincodeDefinition, *pb.Response, []byte, *pb.ChaincodeEvent, error) {
	...
	// 获取ChaincodeInvocationSpec，封装了调用合约的方法和请求参数
	cis, err := putils.GetChaincodeInvocationSpec(txParams.Proposal)
	if err != nil {
		return nil, nil, nil, nil, err
	}

	var cdLedger ccprovider.ChaincodeDefinition
	var version string
	// 检查是否为系统链码
	if !e.s.IsSysCC(cid.Name) {
		// 通过LSCC系统链码获取账本中保存的链码数据对象
		cdLedger, err = e.s.GetChaincodeDefinition(cid.Name, txParams.TXSimulator)
		if err != nil {
			return nil, nil, nil, nil, errors.WithMessage(err, fmt.Sprintf("make sure the chaincode %s has been successfully instantiated and try again", cid.Name))
		}
		version = cdLedger.CCVersion()
		// 检查提案中的实例化合约与调用账本中的实例化合约是否匹配
		err = e.s.CheckInstantiationPolicy(cid.Name, version, cdLedger)
		if err != nil {
			return nil, nil, nil, nil, err
		}
	} else {
		// 获取系统的版本
		version = util.GetSysCCVersion()
	}

	var simResult *ledger.TxSimulationResults
	var pubSimResBytes []byte
	var res *pb.Response
	var ccevent *pb.ChaincodeEvent
	// 启动链码容器调用链码模拟执行
	res, ccevent, err = e.callChaincode(txParams, version, cis.ChaincodeSpec.Input, cid)
	if err != nil {
		endorserLogger.Errorf("[%s][%s] failed to invoke chaincode %s, error: %+v", txParams.ChannelID, shorttxid(txParams.TxID), cid, err)
		return nil, nil, nil, nil, err
	}
	// 获取并处理交易模拟执行结果
	if txParams.TXSimulator != nil {
		if simResult, err = txParams.TXSimulator.GetTxSimulationResults(); err != nil {
			txParams.TXSimulator.Done()
			return nil, nil, nil, nil, err
		}

		if simResult.PvtSimulationResults != nil {
			// 获取因素数据
			pvtDataWithConfig, err := e.AssemblePvtRWSet(simResult.PvtSimulationResults, txParams.TXSimulator)
			txParams.TXSimulator.Done()
			...
			pvtDataWithConfig.EndorsedAt = endorsedAt
			// 分发通道中隐私数据读写集
			if err := e.distributePrivateData(txParams.ChannelID, txParams.TxID, pvtDataWithConfig, endorsedAt); err != nil {
				return nil, nil, nil, nil, err
			}
		}
		txParams.TXSimulator.Done()
		// 获取模拟结果的公有数据
		if pubSimResBytes, err = simResult.GetPubSimulationBytes(); err != nil {
			return nil, nil, nil, nil, err
		}
	}
	return cdLedger, res, pubSimResBytes, ccevent, nil
}
```

callChaincode接口最后调用(cs *ChaincodeSupport) Invoke使用合约服务来模拟执行合约。我们可以看到这个功能函数很简单，只有2个：启动合约容器；执行合约

```go
func (cs *ChaincodeSupport) Invoke(txParams *ccprovider.TransactionParams, cccid *ccprovider.CCContext, input *pb.ChaincodeInput) (*pb.ChaincodeMessage, error) {
    // 1-- 启动合约容器
	h, err := cs.Launch(txParams.ChannelID, cccid.Name, cccid.Version, txParams.TXSimulator)
	if err != nil {
		return nil, err
	}
	cctype := pb.ChaincodeMessage_TRANSACTION
	// 2-- 执行合约
	return cs.execute(cctype, txParams, cccid, input, h)
}
```

###### 1）启动用户链码Docker容器

Launch()接口最后会调用(vm *DockerVM) Start接口创建用户合约容器

```go
// 创建用户合约容器
// 用户链码的容器名称是NetworkID-PeerID-ChaincodeName-ChaincodeVersion
func (vm *DockerVM) Start(ccid ccintf.CCID, args, env []string, filesToUpload map[string][]byte, builder container.Builder) error {
	imageName, err := vm.GetVMNameForDocker(ccid)
	if err != nil {
		return err
	}

	attachStdout := viper.GetBool("vm.docker.attachStdout")
	containerName := vm.GetVMName(ccid)  // 获取镜像名称 
	logger := dockerLogger.With("imageName", imageName, "containerName", containerName)

	client, err := vm.getClientFnc() // 获取容器ID
	if err != nil {
		logger.Debugf("failed to get docker client", "error", err)
		return err
	}

	vm.stopInternal(client, containerName, 0, false, false)
	// 创建指定容器
	err = vm.createContainer(client, imageName, containerName, args, env, attachStdout)
	// 若没有镜像
	if err == docker.ErrNoSuchImage {
		reader, err := builder.Build()
		if err != nil {
			return errors.Wrapf(err, "failed to generate Dockerfile to build %s", containerName)
		}
		// 构建并部署docker 镜像
		err = vm.deployImage(client, ccid, reader)
		if err != nil {
			return err
		}
		// 创建容器
		err = vm.createContainer(client, imageName, containerName, args, env, attachStdout)
		if err != nil {
			logger.Errorf("failed to create container: %s", err)
			return err
		}
	} else if err != nil {
		logger.Errorf("create container failed: %s", err)
		return err
	}
	....
	// 开启容器
	err = client.StartContainer(containerName, nil)
	if err != nil {
		dockerLogger.Errorf("start-could not start container: %s", err)
		return err
	}

	dockerLogger.Debugf("Started container %s", containerName)
	return nil
}
```

用户合约在启动执行时会先调用Peer节点上链码支持服务器的Register()服务接口，请求与Peer侧链码支持服务器建立连接，获取gRPC服务客户端通信流和Peer侧发送与接收消息。链码容器启动流程如下：

![1608715932122](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1608715932122.png)

1）Peer侧调用ChaincodeSupport.Launch()方法启动链码容器。Peer侧调用HandleChaincodeStream()函数与链码容器侧调用chatWithPeer()函数，分别创建两侧对应的Handler对象及其服务状态，建立消息处理循环与通信机制（Golang通道或gRPC通信流）。同时，将两侧服务状态设置为created。链码容器侧构造链码ChaincodeID结构对象chaincodeID，封装了链码名称，并将该对象序列化后封装到ChaincodeMessage_REGISTER类型的链码消息中作为消息负载。然后，调用handler. serialSend()方法，将该消息发送到Peer侧请求注册，将Handler对象（含有与当前链码容器侧连接的通信流）注册到Peer侧的链码服务实例对象上提供服务。

2）Peer侧收到来自链码容器侧的ChaincodeMessage_REGISTER类型消息，交由handler.handleMessage()→Handler.beforeRegisterEvent()方法处理。回复ChaincodeMessage_REGISTERED消息到链码容器侧以通知注册完成。同时将当前状态由created转换为established

3）链码容器侧接收到ChaincodeMessage_REGISTERED消息之后，交由handler.handleMessage()→handler.beforeRegistered()方法进行处理，在检查消息的合法性之后，更新自身状态由created转换为established。

4）然后，Peer侧调用chaincodeSupport.sendReady()→chrte.handler.ready()方法，构造ChaincodeMessage_READY类型的链码消息，首先触发Peer侧的状态由established转换为ready，再将ChaincodeMessage_READY类型的链码消息发送到链码容器侧，通知将状态转换为ready状态，做好链码初始化与调用前的准备工作。至此，链码容器就正式启动完毕。Peer侧与链码容器侧都启动了Handler消息处理句柄与消息处理循环，两侧都处于ready状态，等待执行链码

###### 2）执行链码

(h *Handler) Execute执行链码。执行流程如下图：

Peer侧发送ChaincodeMessage_TRANSACTION类型的链码消息到链码容器侧，请求调用链码的Invoke()方法，并且该链码已经完成了初始化工作。

![1608711117617](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1608711117617.png)



##### 3、处理模拟执行结果

启动链码容器与初始化链码执行环境，模拟执行合法的签名提案消息，并将模拟执行结果记录在交易模拟器中。其中，对公有数据（包含公共数据与隐私数据哈希值）继续签名背书，并提交给Orderer节点请求排序出块，同时将隐私数据通过Gossip消息协议发送到组织内的其他授权节点上。

callChaincode返回执行交易结果，对结果进行处理。通过合法的交易模拟器调用txsim.GetTxSimulationResults()方法，获取当前交易模拟器rwsetBuilder对象保存的模拟执行结果读写集simResult，包括公有数据读写集PubSimulationResults（含有公共数据与隐私数据哈希值）和隐私数据读写集PvtSimulationResults

```go
// 模拟执行提案
func (e *Endorser) SimulateProposal(txParams *ccprovider.TransactionParams, cid *pb.ChaincodeID) (ccprovider.ChaincodeDefinition, *pb.Response, []byte, *pb.ChaincodeEvent, error) {
	...
	// 启动链码容器调用链码模拟执行
	res, ccevent, err = e.callChaincode(txParams, version, cis.ChaincodeSpec.Input, cid)
	if err != nil {
		...
	}
	// 获取并处理交易模拟执行结果
	if txParams.TXSimulator != nil {
		if simResult, err = txParams.TXSimulator.GetTxSimulationResults(); err != nil {
			txParams.TXSimulator.Done()
			return nil, nil, nil, nil, err
		}

		if simResult.PvtSimulationResults != nil {
			...
            // 获取私有数据的读写集
			pvtDataWithConfig, err := e.AssemblePvtRWSet(simResult.PvtSimulationResults, txParams.TXSimulator)
			...
			txParams.TXSimulator.Done()
			...
			// 分发通道中隐私数据读写集，具体分发是gossip协议，本文先不做讨论
			if err := e.distributePrivateData(txParams.ChannelID, txParams.TxID, pvtDataWithConfig, endorsedAt); err != nil {
				return nil, nil, nil, nil, err
			}
		}

		txParams.TXSimulator.Done()
		// 获取模拟结果的公有数据,后续对共有数据进行签名
		if pubSimResBytes, err = simResult.GetPubSimulationBytes(); err != nil {
			return nil, nil, nil, nil, err
		}
	}
	return cdLedger, res, pubSimResBytes, ccevent, nil
}
```

##### 4、对模拟结果签名背书

结果返回到初始的处理函数

```go
func (e *Endorser) ProcessProposal(ctx context.Context, signedProp *pb.SignedProposal) (*pb.ProposalResponse, error) {
	...
	// 1 --模拟执行交易提案
	cd, res, simulationResult, ccevent, err := e.SimulateProposal(txParams, hdrExt.ChaincodeId)
	...
	if chainID == "" {
		// 链ID为空，则不需要背书，直接构造提案响应消息返回
		pResp = &pb.ProposalResponse{Response: res}
	} else {
		// 3 -- 对模拟执行结果进行签名背书
		pResp, err = e.endorseProposal(ctx, chainID, txid, signedProp, prop, res, simulationResult, ccevent, hdrExt.PayloadVisibility, hdrExt.ChaincodeId, txsim, cd)
		...
	}
	// 将处理结果返回给客户端
	pResp.Response = res

	e.Metrics.SuccessfulProposals.Add(1)
	success = true

	return pResp, nil
}
```

背书签名的处理函数endorseProposal，最后调用plugin.Endorse接口对返回的公有数据进行签名，并对数据进行序列化，之后返回给原函数。