### Fabric证书理解

------

/gopath/src/github.com/hyperledger/fabric/fabric-samples/first-network/crypto-config 证书存储路径。下面有很多证书做以解释：

MSP提供的功能：具体的身份格式、用户证书验证、用户证书撤销、签名生成和验证。

sdk访问区块链网络(peer和orderer)会使用tlsca证书

ca.example.com-cert.pem：每个组织会各自生成自己的ca-cert.pem,并注册自己的账号如orderer,peer,user,client,所以每个账号都有自己的cacerts.peer，Org，admin在和orderer传输时，需要的tls证书，是单独的CA Server中颁发。

具体fabric-ca工程发放证书，而在byfn网络是根据脚本生成的证书

```shell
├── ordererOrganizations
│   └── example.com
│       ├── ca
│       │   ├── 461ffe29ee03474d4ac70de6671e424b352117a9ea39b8fa85c99dbfb910f960_sk
│       │   └── ca.example.com-cert.pem
│       ├── msp
│       │   ├── admincerts #管理员证书。目录内的证书代表管理员身份。
│       │   │   └── Admin@example.com-cert.pem
│       │   ├── cacerts #根CA服务器的证书。即为组织颁发证书的CA server目录下的ca-cert.pem
│       │   │   └── ca.example.com-cert.pem
│       │   └── tlscacerts #TLS根CA的证书。即颁发TLS证书的CA server目录下的ca-cert.pem
│       │       └── tlsca.example.com-cert.pem
│       ├── orderers
│       │   └── orderer.example.com
│       │       ├── msp
│       │       │   ├── admincerts
│       │       │   │   └── Admin@example.com-cert.pem
│       │       │   ├── cacerts
│       │       │   │   └── ca.example.com-cert.pem
│       │       │   ├── keystore
│       │       │   │   └── 899ca496f47b80ef0a94d5db86ee8f8db98ba0a430721b6f014f03494e86b6d3_sk
│       │       │   ├── signcerts
│       │       │   │   └── orderer.example.com-cert.pem
│       │       │   └── tlscacerts
│       │       │       └── tlsca.example.com-cert.pem
│       │       └── tls--tls #文件夹中存放加密通信相关的证书文件。一个联盟链中要有一个专门用于颁发传输所需tls证书的CA server
│       │           ├── ca.crt
│       │           ├── server.crt
│       │           └── server.key
│       ├── tlsca
│       │   ├── cdc7c0babc44216cddf1221f949167514d5ae80dac5367fa7b2104d3b45f2606_sk
│       │   └── tlsca.example.com-cert.pem
│       └── users
│           └── Admin@example.com
│               ├── msp
│               │   ├── admincerts
│               │   │   └── Admin@example.com-cert.pem
│               │   ├── cacerts
│               │   │   └── ca.example.com-cert.pem
│               │   ├── keystore
│               │   │   └── 5bc83ae4f39933be66ca17967e336d79779939401f5737372d12cd01d015c365_sk
│               │   ├── signcerts
│               │   │   └── Admin@example.com-cert.pem
│               │   └── tlscacerts
│               │       └── tlsca.example.com-cert.pem
│               └── tls
│                   ├── ca.crt
│                   ├── client.crt
│                   └── client.key
└── peerOrganizations
    ├── org1.example.com
    │   ├── ca #存放组织的根证书和对应的私钥文件，默认采用EC算法，证书为自签名。组织内的实体将基于该证书作为证书根。
    │   │   ├── 2c2a3118261f009c41857a505e3a6191ebf602c5e5b65de380777a9278b9ae1f_sk
    │   │   └── ca.org1.example.com-cert.pem #一个组织的里面所有的ca.***.pem都相同
    │   ├── msp #存放代表该组织的身份信息
    │   │   ├── admincerts #组织管理员的身份验证证书，被根证书签名
    │   │   ├── cacerts #组织的根证书，同ca目录下文件
    │   │   │   └── ca.org1.example.com-cert.pem
    │   │   ├── config.yaml
    │   │   └── tlscacerts #用于TLS的ca证书，自签名
    │   │       └── tlsca.org1.example.com-cert.pem #同一个组织的里面所有的tlsca.***.pem都相同
    │   ├── peers #存放数据该组织的所有peer节点
    │   │   ├── peer0.org1.example.com #包括tls以及msp证书
    │   │   │   ├── msp
    │   │   │   │   ├── admincerts #组织管理员的身份验证证书。peer将基于这些证书来认证交易签署这是否为管理员身份。
    │   │   │   │   ├── cacerts #组织的根证书，同ca目录下文件
    │   │   │   │   │   └── ca.org1.example.com-cert.pem
    │   │   │   │   ├── config.yaml
    │   │   │   │   ├── keystore #本节点的身份私钥，用来签名。
    │   │   │   │   │   └── 04a89eff964eaeb6f8a5de5f657b69c3504f30655b467f4de11c18b13e4eddbc_sk
    │   │   │   │   ├── signcerts # 验证本节点签名的证书，被组织根证书签名。
    │   │   │   │   │   └── peer0.org1.example.com-cert.pem
    │   │   │   │   └── tlscacerts # TLS连接用的身份证书，即组织TLS证书。同msp-tlscacerts-证书
    │   │   │   │       └── tlsca.org1.example.com-cert.pem
    │   │   │   └── tls # 存放tls相关的证书和私钥
    │   │   │       ├── ca.crt # 组织的根证书
    │   │   │       ├── server.crt # 验证本节点签名的证书，被组织根证书签名。
    │   │   │       └── server.key # 本节点的身份私钥，用来签名。
    │   │   └── peer1.org1.example.com #与peer0是相同机构
    │   │       ├── msp
    │   │       │   ├── admincerts
    │   │       │   ├── cacerts
    │   │       │   │   └── ca.org1.example.com-cert.pem
    │   │       │   ├── config.yaml
    │   │       │   ├── keystore
    │   │       │   │   └── 44be821ed3dd14bc734102ea5659601d9be8983ba2c574baf985798e55b6a56e_sk
    │   │       │   ├── signcerts
    │   │       │   │   └── peer1.org1.example.com-cert.pem
    │   │       │   └── tlscacerts
    │   │       │       └── tlsca.org1.example.com-cert.pem
    │   │       └── tls
    │   │           ├── ca.crt
    │   │           ├── server.crt
    │   │           └── server.key
    │   ├── tlsca #存放组织tls连接用的根证书和私钥文件
    │   │   ├── 704a26ca32870971067d4c0bc2d05a9563b76aa4e23262adf2cfd2bcb667421d_sk
    │   │   └── tlsca.org1.example.com-cert.pem
    │   └── users #存放属于该组织的用户实体
    │       ├── Admin@org1.example.com #管理员用户信息，包括msp以及tls
    │       │   ├── msp
    │       │   │   ├── admincerts # 组织管理员的身份验证证书
    │       │   │   ├── cacerts #组织的根证书，同ca目录下文件
    │       │   │   │   └── ca.org1.example.com-cert.pem
    │       │   │   ├── config.yaml
    │       │   │   ├── keystore #本用户的身份私钥，用来签名
    │       │   │   │   └── a2ea8971c8e542edf080003608ea532cff93952911c3c7d8282b6138d9f4cf54_sk
    │       │   │   ├── signcerts # 管理员用户的身份验证证书，被组织根证书签名。要被某个Peer认可，则必须放到该peer的msp/admincerts下。
    │       │   │   │   └── Admin@org1.example.com-cert.pem
    │       │   │   └── tlscacerts # TLS连接用的身份证书，即组织TLS证书
    │       │   │       └── tlsca.org1.example.com-cert.pem
    │       │   └── tls # 存放tls相关的证书和私钥
    │       │       ├── ca.crt # 组织的根证书
    │       │       ├── client.crt# 管理员的用户身份验证证书，被组织根证书签名
    │       │       └── client.key #管理员用户的身份的私钥，用来签名(与/msp/keystore/*sk不同)
    │       ├── User1@org1.example.com #用户，结果同Admin
    │       │   ├── msp
    │       │   │   ├── admincerts
    │       │   │   ├── cacerts
    │       │   │   │   └── ca.org1.example.com-cert.pem
    │       │   │   ├── config.yaml
    │       │   │   ├── keystore
    │       │   │   │   └── 7c71dfcffc08779bfa86cc82063d0ef2ea049b7e8150aead38f689b173e93c29_sk
    │       │   │   ├── signcerts
    │       │   │   │   └── User1@org1.example.com-cert.pem
    │       │   │   └── tlscacerts
    │       │   │       └── tlsca.org1.example.com-cert.pem
    │       │   └── tls
    │       │       ├── ca.crt
    │       │       ├── client.crt
    │       │       └── client.key


```





