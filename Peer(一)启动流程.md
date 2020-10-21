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

(1) 获取本地MSP组件类型

(2) 注册资源访问策略提供者

(3) 初始化本地账本管理器

(4) 初始化服务器基本参数

2、创建gRPC服务器

(1) 创建gRPC服务器

(2) 创建EventHub服务器

(3) 创建DeliverEnvents事件服务器

(4) 创建ChaincodeSupport链码支持服务器

(5) 创建Admin管理服务器与Endorser背书服务器

(6) 创建Gossip消息服务器

3、部署系统链码与初始化现存通道的链结构

(1) 部署系统链码initSysCCs()函数

(2) 初始化现存通道上的链结构Initialize()函数

4、启动gRPC服务器与profile服务器

5、初始化模块日志记录器



