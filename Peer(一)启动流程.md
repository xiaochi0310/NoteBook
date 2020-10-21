### Peer启动流程

------

本文源码解析主要在peer文件夹，如有特殊文件夹会特别标出

main函数在main.go文件中

一、初始化

1、定义主命令

基于Gobra组件构造主命令

```go
var mainCmd = &cobra.Command{
	Use: "peer"}
```

2、注册子命令

```go
func main() {
    ...
    // 设置环境变量前缀core
    viper.SetEnvPrefix(common.CmdRoot)
	viper.AutomaticEnv()
	replacer := strings.NewReplacer(".", "_") // 创建替换符
	viper.SetEnvKeyReplacer(replacer) // 设置环境变量替换符

   // 定义命令行选项集合，对所以peer及子命令都有效
	mainFlags := mainCmd.PersistentFlags()
    // 设置version与logging-level选项
	mainFlags.String("logging-level", "", "Legacy logging level flag")
	viper.BindPFlag("logging_level", mainFlags.Lookup("logging-level"))
	mainFlags.MarkHidden("logging-level")
    // 注册子命令
    mainCmd.AddCommand(version.Cmd())
    mainCmd.AddCommand(node.Cmd())
    mainCmd.AddCommand(chaincode.Cmd(nil))
    mainCmd.AddCommand(clilogging.Cmd(nil))
    mainCmd.AddCommand(channel.Cmd(nil))
	...
}	

```

channel通道子命令：创建应用通道、获取区块、Peer节点加入应用通道、获取节点所加入的应用通道列表、更新应用通道配置、签名配置交易文件、获取指定的应用通道信息等，包括create、fetch、join、list、update、signconfigtx、getinfo等子命令

```go
channelCmd.AddCommand(createCmd(cf))
channelCmd.AddCommand(fetchCmd(cf))
channelCmd.AddCommand(joinCmd(cf))
channelCmd.AddCommand(listCmd(cf))
channelCmd.AddCommand(updateCmd(cf))
channelCmd.AddCommand(signconfigtxCmd(cf))
channelCmd.AddCommand(getinfoCmd(cf))
```

chaincode链码子命令:用于安装链码、实例化（部署）链码、调用链码、打包链码、查询链码、签名链码包、升级链码、获取通道链码列表等，包括install、instantiate、invoke、package、query、signpackage、upgrade、list等子命令

node节点子命令：用于管理节点服务进程与查询服务状态，包括start、status等子命令

logging日志子命令: 用于获取、设置与恢复日志级别功能，包括getlevel、setlevel、revertlevels等子命令；

version版本子命令：打印Hyperledger Fabric中的Peer节点服务器版本信息

```
// byfn展示
```

二、初始化本地MSP组件

在每个子命令中，执行初始化本地MSP组件。

```go
var chaincodeCmd = &cobra.Command{
	Use:   chainFuncName,
	Short: fmt.Sprint(chainCmdDes),
	Long:  fmt.Sprint(chainCmdDes),
	PersistentPreRun: func(cmd *cobra.Command, args []string) {
		common.InitCmd(cmd, args)  // 初始化MSP
		common.SetOrdererEnv(cmd, args)
	},
}
```

MSP组件是管理本地成员身份的重要安全模块，封装了根CA证书、本地签名者实体等。main()函数首先基于Viper组件获取core.yaml文件中MSP组件的配置文件路径mspMgrConfigDir（peer.mspConfigPath配置项）、MSP名称mspID（peer.localMspId配置项）与MSP组件类型mspType（peer.localMspType配置项，默认为FABRIC类型），提供给common.InitCrypto()函数作为参数进行调用，以获取BCCSP区块链加密服务组件，并用于初始化本地MSP组件

```go
// 初始化MSP
func InitCmd(cmd *cobra.Command, args []string) {
    ...
    err := InitConfig(CmdRoot) // 加载core.yaml配置文件
    ...
    // 初始化本地MSP组件对象
	var mspMgrConfigDir = config.GetPath("peer.mspConfigPath")
	// 获取本地MSP名称
    var mspID = viper.GetString("peer.localMspId")
	// 获取本地MSP组件类型
    var mspType = viper.GetString("peer.localMspType")
	if mspType == "" {
        // 默认为FABRIC类型
		mspType = msp.ProviderTypeToString(msp.FABRIC)
	}
    // 获取BCCSCP组件配置信息，初始化MSP组件对象
	err = InitCrypto(mspMgrConfigDir, mspID, mspType)
	...  
}
```

配置文件如下：

![1603099968790](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1603099968790.png)

```go
func InitCrypto(mspMgrConfigDir, localMSPID, localMSPType string) error {
	var err error
	// Check whether msp folder exists
	fi, err := os.Stat(mspMgrConfigDir)
	...
	// 设置BCCSCP密钥存储文件的绝对路径
	SetBCCSPKeystorePath()
	var bccspConfig *factory.FactoryOpts
    // 解析配置文件BCCSCP配置信息保存到变量
	err = viperutil.EnhancedExactUnmarshalKey("peer.BCCSP", &bccspConfig) 
	...
    // 初始化本地MSP组件对象
	err = mspmgmt.LoadLocalMspWithType(mspMgrConfigDir, bccspConfig, localMSPID, localMSPType)
    ...
	return nil
}
```

三、执行启动Peer节点命令

```go
func main() {
	...
	if mainCmd.Execute() != nil {
		os.Exit(1)
	}
    ...
}
```

main()主函数通过Cobra组件调用主命令Execute()方法，执行peer node start命令启动Peer节点

在byfn中调用peer-base.yaml，启动peer node start

![1603101502893](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1603101502893.png)

peer node start 主要流程：主要在start子命令执行server()函数。（函数实现在peer/node/start.go文件）

1、初始化服务启动的基本参数

```go
func serve(args []string) error {
	// (1) 获取本地MSP组件类型，默认基于BCCSCP组件构建的FABRIC类型MSP组件
	mspType := mgmt.GetLocalMSP().GetType()
	if mspType != msp.FABRIC {
		panic("Unsupported msp type " + msp.ProviderTypeToString(mspType))
	}
	
	grpc.EnableTracing = true
	logger.Infof("Starting %s", version.GetInfo())

    // (2) 注册资源访问提供者aclProvider.设置peer以及通道的节点资源的访问权限，例如Lscc_Install不能由peer执行。具体的peer资源对应map为d.pResourcePolicyMap，通道的访问权限对应的map为cResourcePolicyMap
	aclProvider := aclmgmt.NewACLProvider(
		aclmgmt.ResourceGetter(peer.GetStableChannelConfig),
	)

    ...
	// （3）初始化本地账本管理器。详情见下：
	ledgermgmt.Initialize(
		&ledgermgmt.Initializer{
			CustomTxProcessors:            peer.ConfigTxProcessors,
			PlatformRegistry:              pr,
			DeployedChaincodeInfoProvider: deployedCCInfoProvider,
			MembershipInfoProvider:        membershipInfoProvider,
			MetricsProvider:               metricsProvider,
		},
	)

    // (4)初始化服务器基本参数
	if chaincodeDevMode {
		logger.Info("Running in chaincode development mode")
		logger.Info("Disable loading validity system chaincode")
		// 设置链码模式为"dev"
		viper.Set("chaincode.mode", chaincode.DevModeUserRunsChaincode)
	}
    // 读取配置并缓存peer节点地址与端点
	if err := peer.CacheConfiguration(); err != nil {
		return err
	}
	// 获取缓存的peer端点
	peerEndpoint, err := peer.GetPeerEndpoint()
	if err != nil {
		err = fmt.Errorf("Failed to get Peer Endpoint: %s", err)
		return err
	}
    // 获取peer节点的ip地址，为启动节点上的服务器做准备
	peerHost, _, err := net.SplitHostPort(peerEndpoint.Address)
	if err != nil {
		return fmt.Errorf("peer address is not in the format of host:port: %v", err)
	}

```

(1) 获取本地MSP组件类型

代码已经注释

(2) 注册资源访问策略提供者

代码已经注释

(3) 初始化本地账本管理器

peer.ConfigTxProcessors定义通道上配置交易信息处理器字典，且configtxProcessor提供GenerateSimulationResults()方法，用于生成配置交易消息对应的通道配置与资源配置信息，作为模拟执行结果保存到交易模拟器中，再生成模拟执行结果的公共数据，封装成交易对象（Transaction类型）提交给Committer记账节点进行验证

func initialize真正函数

```go
func initialize(initializer *Initializer) {
	logger.Info("Initializing ledger mgmt")
	lock.Lock()
	defer lock.Unlock()
	initialized = true //设置peer节点初始化标志位为true
	openedLedgers = make(map[string]ledger.PeerLedger)
    // 初始化配置交易消息处理器
	customtx.Initialize(initializer.CustomTxProcessors)
    // 初始化链码事件管理器 用于监听处理链码生命周期事件
	cceventmgmt.Initialize(&chaincodeInfoProviderImpl{
		initializer.PlatformRegistry,
		initializer.DeployedChaincodeInfoProvider,
	})
    ...
    // 创建本地peer节点账本提供者 详情见下
	provider, err := kvledger.NewProvider()
    ...
    // 初始化状态监听器
	provider.Initialize(&ledger.Initializer{
        // 初始化Peer节点账本提供者的状态监听器
		StateListeners:                finalStateListeners,
		DeployedChaincodeInfoProvider: initializer.DeployedChaincodeInfoProvider,
		MembershipInfoProvider:        initializer.MembershipInfoProvider,
		MetricsProvider:               initializer.MetricsProvider,
	})
	ledgerProvider = provider
	logger.Info("ledger mgmt initialized")
}
```

创建本地peer账本提供者：kvledger.NewProvider()

1、账本ID数据库（idStore类型）：提供存储账本ID（即链ID）与创世区块键值对的LevelDB数据库；

2、账本数据存储对象提供者（ledgerstorage.Provider类型）：创建账本数据存储对象，负责管理区块数据文件、隐私数据库、区块索引数据库等；

3、历史数据库提供者（HistoryDBProvider类型）：创建历史数据库，存储每个状态数据的历史信息；

4、状态数据库提供者（CommonStorageDBProvider类型）：创建状态数据库（LevelDB或CouchDB类型），存储世界状态（world state），包括有效交易的公有数据与隐私数据

(4) 初始化服务器基本参数

代码已注释

2、创建gRPC服务器

​      peer节点所有的服务器：

![1603269996051](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1603269996051.png)

(1) 创建gRPC服务器

```go
func server() {
	...
    // 获取peer节点的grpc服务器监听地址
	listenAddr := viper.GetString("peer.listenAddress")
	serverConfig, err := peer.GetServerConfig() // 构造grpc服务器的安全配置项
    // 构造gRPC服务器的安全配置项serverConfig。该对象封装了TLS认证所需要的服务器认证证书与密钥、客户端的根CA证书列表、服务器端的根CA证书列表、服务器心跳消息keepalive选项
    ...
    // 创建grpc服务器(7051)
	peerServer, err := peer.NewPeerServer(listenAddr, serverConfig)
	if err != nil {
		logger.Fatalf("Failed to create peer server (%s)", err)
	}
	...
}
```

(2) 创建DeliverEnvents事件服务器

```go
func server() {
	...
    // 若启用TLS安全认证，则配置服务端的根CA证书与客户端证书
	if serverConfig.SecOpts.UseTLS {
		logger.Info("Starting peer with TLS enabled")
		// set up credential support
		cs := comm.GetCredentialSupport()
        // 设置服务器CA证书
		cs.ServerRootCAs = serverConfig.SecOpts.ServerRootCAs
		// 获取gRPC客户端证书用于TLS连接认证
		clientCert, err := peer.GetClientCertificate()
		if err != nil {
			logger.Fatalf("Failed to set TLS client certificate (%s)", err)
		}
		comm.GetCredentialSupport().SetClientCertificate(clientCert)
	}

	mutualTLS := serverConfig.SecOpts.UseTLS && serverConfig.SecOpts.RequireClientCert
	policyCheckerProvider := func(resourceName string) deliver.PolicyCheckerFunc {
		return func(env *cb.Envelope, channelID string) error {
			return aclProvider.CheckACL(resourceName, channelID, env)
		}
	}

	abServer := peer.NewDeliverEventsServer(mutualTLS, policyCheckerProvider, &peer.DeliverChainManager{}, metricsProvider)
	pb.RegisterDeliverServer(peerServer.Server(), abServer)
    ...
}
```

(3) 创建ChaincodeSupport链码支持服务器

Peer节点上的全局链码支持服务实例或链码支持服务器

```go
func server() {
	...
	chaincodeSupport, ccp, sccp, packageProvider := startChaincodeServer(peerHost, aclProvider, pr, opsSystem)
	...
}
```

```go
// startChaincodeServer will finish chaincode related initialization, including:
// 1) setup local chaincode install path
// 2) create chaincode specific tls CA
// 3) start the chaincode specific gRPC listening service
func startChaincodeServer(
	peerHost string,
	aclProvider aclmgmt.ACLProvider,
	pr *platforms.Registry,
	ops *operations.System,
) (*chaincode.ChaincodeSupport, ccprovider.ChaincodeProvider, *scc.Provider, *persistence.PackageProvider) {
	// Setup chaincode path
	chaincodeInstallPath := ccprovider.GetChaincodeInstallPathFromViper()
	ccprovider.SetChaincodesPath(chaincodeInstallPath)

	ccPackageParser := &persistence.ChaincodePackageParser{}
	ccStore := &persistence.Store{
		Path:       chaincodeInstallPath,
		ReadWriter: &persistence.FilesystemIO{},
	}

	packageProvider := &persistence.PackageProvider{
		LegacyPP: &ccprovider.CCInfoFSImpl{},
		Store:    ccStore,
	}

	lifecycleSCC := &lifecycle.SCC{
		Protobuf: &lifecycle.ProtobufImpl{},
		Functions: &lifecycle.Lifecycle{
			PackageParser:  ccPackageParser,
			ChaincodeStore: ccStore,
		},
	}

	// Create a self-signed CA for chaincode service
	ca, err := tlsgen.NewCA()
	if err != nil {
		logger.Panic("Failed creating authentication layer:", err)
	}
	ccSrv, ccEndpoint, err := createChaincodeServer(ca, peerHost)
	if err != nil {
		logger.Panicf("Failed to create chaincode server: %s", err)
	}
	chaincodeSupport, ccp, sccp := registerChaincodeSupport(
		ccSrv,
		ccEndpoint,
		ca,
		packageProvider,
		aclProvider,
		pr,
		lifecycleSCC,
		ops,
	)
	go ccSrv.Start() // 启动服务
	return chaincodeSupport, ccp, sccp, packageProvider
}
```

在peer注册所以系统链码

(4) 创建Admin管理服务器与Endorser背书服务器

Admin管理服务器提供节点启动、节点状态查询等服务。Endorser背书服务器提供对交易提案的模拟执行结果执行签名背书的服务

```go
func server() {
	...
	startAdminServer(listenAddr, peerServer.Server(), metricsProvider)
	// 定义Gossip协议分发隐私数据函数
	privDataDist := func(channel string, txID string, privateData *transientstore.TxPvtReadWriteSetWithConfigInfo, blkHt uint64) error {
		return service.GetGossipService().DistributePrivateData(channel, txID, privateData, blkHt)
	}

	signingIdentity := mgmt.GetLocalSigningIdentityOrPanic()
	serializedIdentity, err := signingIdentity.Serialize()
	if err != nil {
		logger.Panicf("Failed serializing self identity: %v", err)
	}

	libConf := library.Config{}
	if err = viperutil.EnhancedExactUnmarshalKey("peer.handlers", &libConf); err != nil {
		return errors.WithMessage(err, "could not load YAML config")
	}
	reg := library.InitRegistry(libConf)

	authFilters := reg.Lookup(library.Auth).([]authHandler.Filter)
	endorserSupport := &endorser.SupportImpl{
		SignerSupport:    signingIdentity,
		Peer:             peer.Default,
		PeerSupport:      peer.DefaultSupport,
		ChaincodeSupport: chaincodeSupport,
		SysCCProvider:    sccp,
		ACLProvider:      aclProvider,
	}
	endorsementPluginsByName := reg.Lookup(library.Endorsement).(map[string]endorsement2.PluginFactory)
	validationPluginsByName := reg.Lookup(library.Validation).(map[string]validation.PluginFactory)
	signingIdentityFetcher := (endorsement3.SigningIdentityFetcher)(endorserSupport)
	channelStateRetriever := endorser.ChannelStateRetriever(endorserSupport)
	pluginMapper := endorser.MapBasedPluginMapper(endorsementPluginsByName)
	pluginEndorser := endorser.NewPluginEndorser(&endorser.PluginSupport{
		ChannelStateRetriever:   channelStateRetriever,
		TransientStoreRetriever: peer.TransientStoreFactory,
		PluginMapper:            pluginMapper,
		SigningIdentityFetcher:  signingIdentityFetcher,
	})
	endorserSupport.PluginEndorser = pluginEndorser
	serverEndorser := endorser.NewEndorserServer(privDataDist, endorserSupport, pr)
	...
}
```



(5) 创建Gossip消息服务器

3、部署系统链码与初始化现存通道的链结构

(1) 部署系统链码initSysCCs()函数

(2) 初始化现存通道上的链结构Initialize()函数

4、启动gRPC服务器与profile服务器

5、初始化模块日志记录器



