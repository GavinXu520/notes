# Fabric CA 模块

------

## 目录

- [概述](#概述)
- [环境](#环境)
  - [系统设置](#系统设置)
  - [依赖包](#依赖包)
  - [Go](#go)
  - [Fabric-CA](#fabric-ca)
- [配置](#配置)
  - [Server配置](#server配置)
    - [环境变量](#环境变量)
    - [初始化Server](#初始化server)
    - [编辑Server配置文件](#编辑server配置文件)
    - [启动Server](#启动server)
  - [Client配置](#client配置)
    - [登记admin身份](#登记admin身份)
- [操作](#操作)
  - [注册身份](#注册身份)
  - [登记身份](#登记身份)
  - [相关说明](#相关说明)
    - [admin身份说明](#admin身份说明)
    - [注册和登记说明](#注册和登记说明)
  - [重新登记身份](#重新登记身份)
  - [撤销身份或证书](#撤销身份或证书)
  - [获取CA证书链](#获取ca证书链)
  - [启用TLS](#启用tls)
  - [中间CA](#中间ca)
    - [基础知识](#基础知识)
    - [创建中间CA服务器](#创建中间ca服务器)
  - [说明](#说明)
- [结语](#结语)

------

## 概述

Fabric-CA是Fabric v1.0版以后代替v0.6版中的Membership Service的用于生成和管理节点证书和密钥的CA(证书颁发机构)服务模块, 其主要功能包括身份注册, 身份登记, 身份更改, 身份撤销, 以及证书签发,续期与撤销, Fabric-CA包括Server端和Client端, Server端负责提供身份证书密钥等服务, Server端可以由多个CA服务器组成CA集群, 通过HA Proxy负载均衡对外提供服务. Client端负责与Server端进行交互, 发起服务请求, 另一种与Server端进行交互的方式是Fabric SDK.

Fabric-CA的架构和在Fabric网络中所处角色如下图所示:
![Fabric-CA架构图](./images/fabric-ca.png)

------

## 环境

### 系统设置

设置主机名, 关闭防火墙, 设置主机名-IP映射.

```bash
systemctl stop firewalld                        #开启防火墙,只开放fabric-ca的7054端口也可
systemctl disable firewalld
hostnamectl set-hostname ca.example.com         #无关紧要
vim /etc/hosts
    ......
    10.0.2.11    ca.example.com
```

### 依赖包

用于编译相关源码.

```bash
yum install snappy-devel zlib-devel bzip2-devel libtool libtool-ltdl-devel -y
```

### Go

版本要求 v1.9+, 官网下载, 用于编译fabric-ca源码, 生成fabric-ca-client和fabric-ca-server.

```bash
wget https://studygolang.com/dl/golang/go1.10.linux-amd64.tar.gz
tar -zxvf go1.10.linux-amd64.tar.gz -C /usr/local/share/    #解压
mkdir -p /root/go         #创建go本地运行目录
vim /etc/profile          #创建go环境变量
    ......
    #Go
    export GOROOT=/usr/local/share/go
    export GOPATH=/root/go
    export PATH=$GOROOT/bin:$GOPATH/bin:$PATH
source /etc/profile
```

### Fabric-CA

用于生成fabric-ca-client和fabric-ca-server.

```bash
go get -u github.com/hyperledger/fabric-ca/cmd/...        #相当于git clone && go install
```

> 说明:
> - 此命令会将fabric-ca源码下载到$GOPATH/src/github.com/hyperledger/目录下, 并安装fabric-ca-client和fabric-ca-server到$GOPATH/bin目录下
> - 如果生成的fabric-ca-client和fabric-ca-server执行时出现"Version is not set for fabric-ca library"错误, 则直接编译fabric-ca源码(make), 将$GOPATH/src/github.com/hyperledger/fabric-ca/build/docker/bin目录下生成的fabric-ca-client和fabric-ca-server文件复制到$GOPATH/bin目录下, 然后删除fabric-ca源码即可.

------

## 配置

### Server配置

#### 环境变量

建议先设置ca-server的HOME目录环境变量, HOME目录优先级:

1. FABRIC_CA_SERVER_HOME
2. FABRIC_CA_HOME
3. CA_CFG_PATH
4. 当前工作目录

```bash
echo "export FABRIC_CFG_PATH=/etc/hyperledger/fabric" > /etc/profile
echo "export FABRIC_CA_HOME=$FABRIC_CFG_PATH/ca" > /etc/profile
echo "export FABRIC_CA_SERVER_HOME=$FABRIC_CA_HOME/ca-server" > /etc/profile      #ca-server HOME目录
echo "export FABRIC_CA_CLIENT_HOME=$FABRIC_CA_HOME/ca-client" > /etc/profile      #ca-client HOME目录
source /etc/profile
```

#### 初始化Server

```bash
fabric-ca-server init -b admin:adminpw
```

> 说明:
> - 不使用LDAP的时候, ca-server需要有一个预注册的引导身份(pre-registered bootstrap identity), 用于注册(register)和登记(enroll)其他的身份, 初始化的时候通过-b参数指定引导身份(也即引导管理员 bootstrap admin)的账号和密码
> - 初始化后会在$FABRIC_CA_SERVER_HOME目录下生成四个文件：
>   - fabric-ca-server-config.yaml: 默认ca-server配置文件
>   - ca-cert.pem: 自我认证的CA证书文件
>   - fabric-ca-server.db: 默认的CA服务器数据库(sqlite3）文件
>   - msp/keystore/xxx_sk: 自我认证的密钥文件(私钥)
> - ca-cert.pem是自签名证书, 它本身也是根据fabric-ca-server-config.yaml配置文件的内容生成的, 可以使用openssl查看证书文件的内容(openssl自带ca服务器模拟功能), 命令如下：
>> openssl x509 -in ca-cert.pem -inform pem -noout -text    #可以看到签名算法, 证书颁发者的affiliation, 申请者/所有者的affiliation, 申请者/所有者的公钥, 起止有效期等信息
> - 如果ca-server是多节点集群的模式,那么fabric-ca-server数据库必须是postgres或mysql数据库,同时向所有ca节点服务；如果ca-server是单节点,那么fabric-ca-server数据库默认是本地sqlite3数据库。ca数据库中主要有三个表(直接使用sqlite3命令行工具查看表结构和表数据)：
>   - affiliations: 描述组织结构层次关系(默认存储了默认的fabric-ca-server-config.yaml配置文件中的affiliations内容)
>   - certificates: 颁发的所有证书（默认是空表）
>   - users: ca-server注册的所有用户(默认存储了默认的fabric-ca-server-config.yaml配置文件中的identities内容), 这里的用户指包括客户端client,peer节点,orderer节点,user用户在内的所有注册类型

#### 编辑Server配置文件

默认的fabric-ca-server-config.yaml配置文件中包含对ca-server服务本身的配置和默认CA的配置.

> 三种方法可以配置ca-server(优先级由大到小):
> - CLI命令行(eg. fabric-ca-client enroll --tls.client.certfile cert3.pem)
> - 环境变量(eg. export FABRIC_CA_CLIENT_TLS_CLIENT_CERTFILE=cert2.pem)
> - 配置文件
> 一般使用配置文件进行参数配置, 配置文件的配置可以被环境变量和命令行参数覆盖, CLI或环境变量未设置的参数则使用配置文件中的值

```yaml
version:                                                      #配置文件版本
port: 7054                                                    #ca-server监听端口
debug: false                                                  #是否开启debug日志
crlsizelimit: 512000                                          #证书吊销列表的大小限制(bytes)
tls:                                                          #ca-server监听端口的tls加密传输配置(即server和client数据传输是否加密), 此时节点是作为serer的
  enabled: false                                              #默认不启用server和client的传输加密
  certfile:                                                   #ca-server的证书
  keyfile:                                                    #ca-server的密钥
  clientauth:
    type: noclientcert                                        #对客户端的认证要求(NoClientCert, RequestClientCert, RequireAnyClientCert, VerifyClientCertIfGiven, RequireAndVerifyClientCert)
    certfiles:                                                #ca-server用于验证客户端证书的一系列受信的CA证书文件
ca:                                                           #此ca-server的CA证书和密钥配置, 通过指定keyfile和certfile可以选择使用另外指定的证书和密钥文件,而不使用自动生成的
  name: fabric-ca-server                                      #证书颁发机构的名字, 对一个区块链网络中的所有成员是唯一的
  keyfile:                                                    #私钥文件, 用于将私钥导入BCCSP(blockchain crypto service provider)中
  certfile:                                                   #CA自己的证书(包括登记证书ECerts和交易证书TCerts)
  chainfile:                                                  #CA受信的证书链文件
crl:                                                          #证书吊销列表的配置
  expiry: 24h                                                 #更新证书吊销列表的间隔时间(即CRL的过期时间)

#############################################################################
#  注册部分的配置主要用于控制ca-server处理以下两个任务:
#  1) 对包含用户名和密码(即登记ID和口令)的登记请求进行认证
#  2) 一旦认证成功, 将检索登记者的相关属性键值, ca-server会将(可选的)这些内容写入TCerts(TCert用于提供区块链交易时的匿名性和不可链接性), 这些属性对于在链码中进行访问权限控制非常有用
#  当下文中ldap部分的ldap.enabled为false时, registry部分才会生效, 此时由ca-server来处理上述的登记和登记者的身份属性检索, 如果ldap.enabled为true时, 则由下文的ldap部分来配置并由ca-server授权ldap服务器来完成这两项功能, registry部分的配置失效
#############################################################################
registry:                                                  #认证和检索配置
  maxenrollments: -1                                       #用于登记的密码/口令最大可重复使用的次数(即有多少用户可以使用相同的密码), -1为无限制, 0为不允许任何用户登记
  identities:                                              #ca-server上注册的身份信息
     - name: admin                                         #ldap禁用的情况下, ca-server至少需要一个预设身份, 默认是admin
       pass: adminpw
       type: client                                        #client类型
       affiliation: ""                                     #预设组织关系, 非空时, 新注册的身份必须是该预设关系的前缀(如affiliation为"a.b.c", 则注册身份时的--id.affiliation可以是a.b, 但不可以是a.c), 相当于affiliation描述了一个ca-server可以处理的完整的组织机构的上下级部门结构
       attrs:                                              #身份的属性
          hf.Registrar.Roles: "peer,orderer,client,user"   #身份可以管理的注册角色
          hf.Registrar.DelegateRoles: "peer,orderer,client,user"    #身份注册的角色可以管理的角色(即该身份注册的角色的hf.Registrar.Roles的允许值)
          hf.Revoker: true                                 #身份是否可以移除身份或者证书
          hf.IntermediateCA: true                          #身份是否是中间CA, 即是否可以作为其他ca-server的父server(可以为其他ca-server签发CA证书)
          hf.GenCRL: true                                  #身份是否可以创建CRL证书吊销列表
          hf.Registrar.Attributes: "*"                     #身份注册其他身份时允许指定的属性(*表示所有属性均可注册)
          hf.AffiliationMgr: true                          #身份是否可以管理组织关系
db:                                                        #ca-server记录注册认证相关信息(组织关系, 证书, 用户等)的数据库配置
  type: sqlite3                                            #默认使用sqlite3数据库, 但此数据库属于内嵌数据库(无法提供远程访问), 因此如果需要部署多台ca-server(即CA cluster), 则必须指定为postgres或者mysql数据库
  datasource: fabric-ca-server.db                          #不同数据库有不同的连接代码, 如远程postgres数据库的连接代码:host=localhost port=5432 user=Username password=Password dbname=fabric_ca sslmode=verify-full
  tls:                                                     #数据库连接和数据传输是否使用tls加密
      enabled: false                                       #如果启用, 那么数据库端也要作ssl相关设置
      certfiles:                                           #PEM编码的可信CA证书文件列表(证书颁发机构给自己颁发的证书)
      client:                                              #对于数据库而言, ca-server是client, 数据库是server
        certfile:                                          #ca-server的PEM编码的证书和密钥文件
        keyfile:
#cluster数据库的配置详见:http://hyperledger-fabric-ca.readthedocs.io/en/latest/users-guide.html
ldap:                                                     #ldap配置, 启用后, 认证和检索身份的工作将由ldap接管
   enabled: false
   url: ldap://<adminDN>:<adminPassword>@<host>:<port>/<base>
   tls:
      certfiles:
      client:
         certfile:
         keyfile:
   attribute:
      names: ['uid','member']
      converters:
         - name:
           value:
      maps:
         groups:
            - name:
              value:
affiliations:                                            #组织关系, 描述现实中组织的组成结构, 非叶节点都是大小写不敏感的, 默认都是小写
   org1:
      - department1
      - department2
   org2:
      - department1
signing:                                                #签名部分
    default:                                            #签发登记证书
      usage:
        - digital signature
      expiry: 8760h                                     #默认过期时间1年
    profiles:
      ca:                                               #签发中间CA证书(即为其他ca-server签发证书)
         usage:
           - cert sign
           - crl sign
         expiry: 43800h                                 #默认过期时间5年
         caconstraint:
           isca: true                                   #表明证书是CA证书
           maxpathlen: 0                                #表明使用其签发的CA证书的ca-server继续作为中间CA给其他CA签发CA证书的可迭代层数(0表示使用其签发的CA证书的ca-server不能继续作为中间CA给其他CA签发CA证书)
      tls:                                              #签发TLS证书
         usage:
            - signing
            - key encipherment
            - server auth
            - client auth
            - key agreement
         expiry: 8760h                                  #默认过期时间1年
csr:                                                    #证书请求文件, 即CA证书的配置, 会由fabric-ca-server命令根据此配置自动生成此ca-server的CA证书, 如果不需要自动生成, 可以在上面ca配置部分指定certfile和keyfile的路径, 并在相应路径下存放自己提供的证书和密钥
   cn: fabric-ca-server
   key:                                                 #加密算法
      algo: ecdsa
      size: 256
   names:
      - C: US                                           #国家
        ST: "North Carolina"                            #州
        L:                                              #城市或位置
        O: Hyperledger                                  #组织
        OU: Fabric                                      #部门
   hosts:                                               #证书在哪些主机名的主机上可用
     - ca
     - localhost
   ca:
      expiry: 131400h                                   #默认过期时间15年
      pathlength: 1                                     #同上
bccsp:                                                  #区块链加密服务提供者
    default: SW                                         #使用基于软件的区块链加密服务
    sw:
        hash: SHA2                                      #加密算法
        security: 256
        filekeystore:
            # The directory used for the software file-based keystore
            keystore: msp/keystore
cacount:                                                      #每个ca-server默认保含一个CA, 也可以配置多个

cafiles:                                                      #每个CA可以有独立的配置文件, 除了port, debug和tls部分(这些是与服务器相关的配置)外, 其他在默认CA配置文件中的配置项都可以在CA独立的配置文件中配置, 每个CA配置文件中都必须指定唯一的ca.name和csr.cn.

#############################################################################
# 中间CA部分
# ca-server和CA的关系如下:
#   1) 一个服务进程可以包含一个或多个CA.
#   2) CA分根CA和中间CA两种
#   3) 每个中间CA都有一个父CA, 这个父CA要么是根CA, 要么是另外一个中间CA.
#############################################################################
intermediate:                                            #中间CA的参数配置
  parentserver:                                          #指定中间CA的父CA
    url:                                                 #父CA的服务地址
    caname:                                              #父CA的唯一name

  enrollment:                                            #向父CA登记的信息
    hosts:
    profile:
    label:

  tls:                                                   #安全套接字连接(tls加密传输)
    certfiles:                                           #受信CA证书列表
    client:
      certfile:
      keyfile:
```

#### 启动Server

根据配置好的fabric-ca-server-config.yaml配置文件启动ca-server, 事实上如果不对默认配置作更改, 或者直接在命令行中使用配置参数进行配置, 可以不需要初始化, 直接启动server.

```bash
fabric-ca-server start
fabric-ca-server start -b admin:adminpw                                   #如果不进行初始化直接启动需要指定引导身份
```

### Client配置

在ca-client节点登记admin引导身份(即引导管理员), 登记(enroll)操作即从ca-server获取一个身份(该身份存在于ca-server数据库中)作为该节点的身份, 之前配置的admin身份是client类型, 因此该节点就成为一个具有admin身份的相关权限和属性的client节点, 然后在此节点上向ca-server节点发起身份注册请求. 本例server和client节点为同一台服务器。

#### 登记admin身份

```bash
fabric-ca-client enroll -u http://admin:adminpw@10.0.2.11:7054            #10.0.2.11为server节点IP
```

> 说明:
> - 命令执行完成会在$FABRIC_CA_CLIENT_HOME目录下生成一个msp目录和一个默认的fabric-ca-client-config.yaml配置文件, msp目录下包含四个目录：
>   - keystore/xxx_sk: 本节点的私钥
>   - signcerts/cert.pem: 本节点的数字证书(ECert), 这是由ca-server上的CA自动签发的
>   - cacerts/10-0-2-11-7054.pem: CA证书文件, 格式为enroll时的指定的ca-server的IP/域名+端口, 此文件其实就是ca-server上的CA证书, CA证书中包含CA的公钥, 通过此公钥可以验证本节点的数字证书是否是CA颁发的(CA签发的证书是经过CA的公钥加密的, 如果节点的数字证书能够使用CA证书中的公钥解密, 则表示证书合法)
>   - intermediatecerts/10-0-2-11-7054.pem: 中间CA的证书
> - 命令执行完之后可以在ca-server的fabric-ca-server.db数据库的certificates表中看到一条记录, 即ca-server上的CA为admin身份颁发的一个证书

默认的fabric-ca-client-config.yaml配置文件内容如下(修改内容后可以通过fabric-ca-client reenroll命令重新根据此配置文件的内容进行登记)：

```yaml
url: http://10.0.2.11:7054                              #ca-server的服务IP和端口
mspdir: msp                                             #节点登记身份后生成的相关认证相关文件存放的目录(相对路径)
tls:                                                    #tls数据加密传输配置, 此时本节点是作为client的
  certfiles:                                            #用于验证自身证书的一系列受信的CA证书文件
  client:
    certfile:                                           #节点自身的证书
    keyfile:                                            #节点自身的密钥
csr:                                                         #证书请求文件, 即节点证书的配置, ca-server根据收到的此信息生成节点的证书
  cn: admin                                                  #cn值必须与节点登记的身份的ID值一致
  serialnumber:
  names:
    - C: US
      ST: North Carolina
      L:
      O: Hyperledger
      OU: Fabric
  hosts:                                                     #证书在哪些主机名的主机上可用
    - ca
id:                                                          #向ca-server注册身份的配置
  name:                                                      #身份的唯一识别名(ID)
  type:                                                      #身份的类型
  affiliation:                                               #身份所属的组织
  maxenrollments: 0                                          #身份的密码/口令最多的重用次数(即多少个身份可以使用相同的密码), 0表示使用ca-server的设置, 这个值必须小于等于ca-server的配置值
  attributes:                                                #身份的一系列属性
   # - name:
   #   value:
enrollment:                                                  #登记的身份
  profile:                                                   #签名配置文件
  label:                                                     #分层存储管理(即fabric中的区块链加密管理套件bccsp)
caname:                                                      #连接ca-server上的哪个CA
bccsp:                                                       #节点使用的加密管理套件
    default: SW                                              #基于软件的加密实现
    sw:
        hash: SHA2                                           #加密算法
        security: 256
        filekeystore:
            keystore: msp/keystore                           #密钥存储位置

```

------

## 操作

完成上述配置后, ca-server节点已经可以进行证书的签发, ca-client节点已经拥有引导管理员的身份, 可以向ca-server提出注册,登记等相关认证请求了. 接下来在client节点上执行相关操作, 进一步理解Fabric ca的工作机制.
> 在client节点上执行注册和撤销操作要求client节点必须是具有相应权限(即相关属性)的身份, 引导管理员(bootstrap admin)身份的默认配置已经保证该身份拥有最高的权限

### 注册身份

当前ca-server中只有一个引导管理员身份, 现在尝试向ca-server中注册(register)一个身份.

```bash
fabric-ca-client register --id.name adm --id.type user --id.secret admpw --id.affiliation org1.department1 --id.attrs 'hf.Revoker=true,foo=bar'
```

> 说明：
> - 此命令注册了一个名为adm, 类型为user, 组织关系为org1.department1, 属性为"hf.Revoker=true,foo=bar"的身份
> - 此命令执行过程中, 对于命令行参数未指定的参数项, 会默认使用fabric-ca-client-config.yaml配置文件的配置值
> - 此命令执行完成会返回该身份的密码, 如果不指定--id.secret, ca-server会生成一个随即密码, 使用该身份的身份名(enrollment ID)和密码(secret)可以在其他节点登记(enroll)该身份
> - 此命令执行完成会在ca-server的fabric-ca-server.db数据库的users表中新增一条记录
> - 注册过程中, ca-server会做相关检查, 只有所有检查均通过, 身份注册才能成功. 检查内容如下：
>   - id.name: 需要注册的身份的name必须是唯一的, 不能与ca-server的fabric-ca-server.db数据库中已存在的项重复
>   - id.type: 需要注册的身份的type必须是client节点的身份(当前是admin引导身份)的hf.Registrar.Roles属性中指明的可以管理的注册身份类型
>   - id.secret: 手动指定新注册身份的密码, 此密码必须满足该身份的maxenrollments和ca-server的maxenrollments属性的重用次数要求
>   - id.affiliation: 需要注册的身份的组织关系必须是client节点的身份(当前是admin引导身份)的affiliation指明可以处理的组织关系的前缀(如果当前client节点的身份的affiliatin值为org1.department1.team1, 那么注册的身份的组织关系可以是org1, org1.department1或org1.department1.team1, 而不可以是 org1.team1), 另外, 需要注册的身份的组织关系也必须是ca-server的配置文件中affilications项所描述的组织关系(通常身份的affiliation中的组织关系是ca-server配置文件的affiliations项的组织关系的子集)
>   - id.attrs: 需要注册的身份的属性必须是client节点的身份(当前是admin引导身份)的hf.Registrar.Attributes属性中指明可以处理的属性

### 登记身份

在另外一个节点上登记(enroll)刚刚注册的身份(adm), 登记操作会为该身份在节点上生成认证相关文件(与上文登记admin身份一样), 如果节点上之前已经登记了一个身份, 那么新的身份会顶替掉旧的身份(旧身份的认证文件都会被新文件覆盖掉).

```bash
export FABRIC_CA_CLIENT_HOME=/etc/hyperledger/fabric/ca/ca-client
fabric-ca-client enroll -u http://adm:admpw@10.0.2.11:7054
```

> 说明:
> - 所有需要登记身份的节点建议在登记身份前都设置FABRIC_CA_CLIENT_HOME环境变量, 登记过程中产生的认证相关文件会保存在此环境变量目录中, 如未指定Fabric ca相关环境变量, 生成的认证相关文件默认会存放在执行命令的当前目录中
> - 登记成功后会在$FABRIC_CA_CLIENT_HOME目录下(同样依环境变量优先级而定)生成msp目录并将认证相关文件(MSP material)存放在msp目录中, 也可以在命令行使用-M手动指定认证相关文件的存放位置
> - 登记操作同样会读取fabric-ca-client-config.yaml配置文件中的配置信息, 可以在enroll之前先修改文件中的csr部分, 实现证书信息的自定义
> - 登记操作会自动将登记请求发往默认的CA, 也可以在命令行中(--caname)或者fabric-ca-client-config.yaml配置文件中(caname)手动指定CA
> - 本例中ca-server和ca-client是同一台节点服务器, 因此如果在该节点上登记了adm身份, 将无法再进行身份注册操作(因为adm身份没有hf.Registrar.Roles等相关属性), 可以再次enroll admin身份以重新使用admin身份执行操作

### 相关说明

#### admin身份说明

ca-server启动时默认是指定了一个引导身份(admin:adminpw)的, 拥有引导身份的节点是可以发起并正常执行身份注册和撤销请求的. 上文说过, 注册和撤销操作只要是拥有相应属性的身份都可以执行, 而无须一定是初始的admin引导身份. 现在就通过注册一个与admin拥有同样属性的身份, 并登记该身份, 验证此身份是否能正常执行身份注册和撤销请求.

本例中不再在命令行参数中指定身份的相关属性, 而是直接在fabric-ca-client-config.yaml配置文件中修改id项, 然后直接注册.

- 编辑client配置文件

```yaml
#只改变id项的配置, 其他不变
id:
  name: ad                                  #注意这里指定pass项的话也是无效的, fabric-ca-client还是会为身份生成随机password
  type: client                              #并不要求type一定是client
  affiliation: ""
  maxenrollments: 0
  attributes:                              #这里指定属性的语法不一样
    - name: "hf.Registrar.Roles"
      value: "peer,orderer,client,user"    #这个属性的值决定了该身份是否可以注册相应的身份类型
    - name: "hf.Registrar.DelegateRoles"
      value: "peer,orderer,client,user"
    - name: "hf.Revoker"
      value: true                          #这个属性的值决定了该身份是否可以移除身份或者证书
    - name: "hf.IntermediateCA"
      value: true
    - name: "hf.GenCRL"
      value: true                          #这个属性的值决定了该身份是否可以创建CRL证书吊销列表
    - name: "hf.Registrar.Attributes"
      value: "*"                           #这个属性的值决定了使用该身份注册其他身份时允许指定的属性(*表示所有属性均可注册)
    - name: "hf.AffiliationMgr"
      value: true                          #这个属性的值决定了该身份是否可以管理组织关系)
```

- 注册身份

不指定身份相关属性直接注册:

```bash
fabric-ca-client register
```

- 登记身份

```bash
fabric-ca-client enroll -u http://ad:kBiAFPhutHWZ@10.0.2.11:7054            #kBiAFPhutHWZ是注册时自动生成的密码
```

- 使用该身份注册另一个身份

```bash
fabric-ca-client register --id.name addm --id.type user --id.affiliation org1.department1 --id.attrs 'hf.Revoker=true,foo=bar'
```

发现可以注册成功

- 使用该身份撤销admin引导管理员身份

```bash
fabric-ca-client revoke -e admin
```

发现admin身份可以被成功撤销(撤销身份不会将身份从ca-server数据库中删除,只会删除该身份的所有证书并阻止该身份再得到新的证书, 即阻止其被enroll), 再次enroll admin身份会失败

- 结论

Fabric CA模块中ca-server的初始引导身份admin并非特殊身份(身份name也可以是任意的), 该身份的唯一作用是作为一个默认的权限完整的自证身份在某节点上进行登记然后实现其他身份的注册和撤销等操作.

#### 注册和登记说明

- 注册(register)是向ca-server提出生成新身份的请求, 需要提供待注册身份的身份名, 密码, 类型, 组织关系以及各种属性信息(命令行参数或者yaml配置文件方式均可提供相关信息). 发起注册请求要求节点当前登记的身份必须要具有与注册相关的属性(hf.Registrar.Roles)才可能注册成功. 注册过程中不会生成本地文件, 但会在ca-server数据库中的users表新增一条记录.
- 登记(enroll)是将ca-server中已存在的身份登记到本地节点, 一个身份可以同时登记到多个不同节点, 一个节点同时只能登记一个身份. 登记新的身份会导致旧身份被覆盖(不会影响ca-server上的身份本身). 登记操作对节点本身没有要求(即只要有fabric-ca-client命令的节点都可以登记一个身份), 会在节点本地生成与认证相关的安全文件(可以在yaml配置文件的csr配置项中自定义证书信息). 每一次登记操作都会在ca-server数据库中的certificates表中新增一条记录.

### 重新登记身份

如果登记了某个身份的节点上的证书过期了, 或者需要修改csr相关的证书信息, 可以再次执行一次enroll命令, 也可以直接重新登记(reenroll), reenroll可以避免输入冗长的登记参数或者身份登记错乱.

```bash
fabric-ca-client reenroll
```

### 撤销身份或证书

Fabric CA模块的撤销操作支持撤销身份的某些属性, 撤销身份的某些证书, 甚至直接撤销身份. 撤销证书只会导致一个证书无效, 撤销身份会将身份的全部证书都撤销, 并阻止其再得到新的证书, 即该身份不再可用.

```bash
fabric-ca-client revoke -e addm                                          #撤销身份

serial=$(openssl x509 -in msp/signcerts/cert.pem -serial -noout | cut -d "=" -f 2)
aki=$(openssl x509 -in msp/signcerts/cert.pem -text | awk '/keyid/ {gsub(/ *keyid:|:/,"",$1);print tolower($0)}')
fabric-ca-client revoke -s $serial -a $aki -r affiliationchange          #撤销证书
```

> 说明:
> - 无论是撤销证书还是撤销身份都不会将ca-server数据库中的相应记录删除, 而只是修改相应记录的状态字段
>   - certifiates表中status字段值为good表示证书当前可用, 值为revoked表示证书已被撤销
>   - users表中state字段值表示该身份已被登记的次数(即有效证书的个数), 值为-1表示该身份被撤销, 不可再得到新的证书
> - 执行身份撤销操作的身份必须具有hf.Revoker属性且值为true, 同时还必须具有hf.Registrar.Roles属性, 保证身份被撤销后至少有一个身份还能执行注册操作
> - 可以在身份或证书撤销时使用-r指定撤销原因, 可选的原因有:
>   - unspecified
>   - keycompromise
>   - cacompromise
>   - affiliationchange
>   - superseded
>   - cessationofoperation
>   - certificatehold
>   - removefromcrl
>   - privilegewithdrawn
>   - aacompromise

### 获取CA证书链

通常一个ca-server上可以有多个CA, 多个CA均可为末端节点(orderer, peer, client, user)签发证书, 也就是说不同节点的证书可能处于不同的根信任分支中, 为了节点间能够互相验证证书的合法性, 需要在各个节点上将各个CA的CA证书加入其本地存放CA证书的目录中.

```bash
fabric-ca-client getcacert -u http://10.0.2.11:7054                          #t同样可以使用-M选项手动指定证书存放位置
```

### 使用TLS

可以为fabric-ca-client和fabric-ca-server之间的通信启用TLS套接字加密传输, 需要在ca-server和ca-client的yaml配置文件中分别启用TLS

```yaml
# fabric-ca-server-config.yaml
tls:
  # Enable TLS (default: false)
  enabled: true
  certfile: ca-cert.pem                                       #ca-server自身的证书
  keyfile: xxx_sk                                             #ca-server自身的密钥
  clientauth:
    type: requireclientcert                                   #对客户端的认证要求
    certfiles:                                                #ca-server用于验证客户端证书的一系列受信的CA证书文件
      - ca-cert.pem
```

```yaml
# fabric-ca-client-config.yaml
tls:
  # Enable TLS (default: false)
  enabled: true
  certfiles:                                                 #一系列受信的CA证书文件
    - ca-cert.pem
  client:                                                    #client节点自身的证书和密钥
    certfile: cert.pem
    keyfile: xxx_sk
```

### 中间CA

#### 基础知识

单节点的ca-server存在单点故障的问题, 因此生产上通常部署ca-server集群, 最开始的ca-server上的CA会为CA集群中的节点签发CA证书, 最初的这个CA(自签名CA)称为根CA(root CA), 由其他CA签发CA证书的CA称为中间CA(intermediate CA), 根CA和中间CA都可以为其他CA签发证书, 即CA之间存在数状层次关系, 所有非根CA统称中间CA.

中间CA是根CA的代理, 其证书由根CA或其他中间CA签发, 中间CA也能代表根CA为其他CA或末端用户节点签发证书. 中间CA向根CA申请CA证书时, 不仅会获得自己的CA证书, 私钥, 还会获得一个证书链(pem文件), 从根CA到当前节点(不限于CA节点)的数状分杈上的所有证书都可以被追加到证书链上, 因此只需要利用证书链即可回溯验证链上每一个证书的合法性. 即如果根CA证书是受信的, 那么此根CA的证书链上的所有证书都是受信的.

Fabric的CA集群使用HA Proxy实现负载均衡, 但通常根CA服务器并不参与负载均衡, 事实上, 根CA服务器甚至可以直接下线, 只需要保留根CA的证书和密钥即可, 只有在撤销或更改中间CA的证书的时候才需要根CA的证书和密钥.

#### 创建中间CA服务器

创建并启动中间CA只需要一条命令:

```bash
fabric-ca-server start -b admin:adminpw -u http://admin:adminpw@10.0.2.11:7054           #只需要指定父CA即可
```

> 说明:
> - 也可以像根CA的创建和启动一样,先初始化, 修改配置文件, 然后启动:
>   - fabric-ca-server init -b admin:adminpw -u http://admin:adminpw@10.0.2.11:7054
>   - fabric-ca-server start
> - 命令执行完成后会在FABRIC_CA_SERVER_HOME目录下生成5个文件/目录:
>   - ca-cert.pem: 中间CA的CA证书
>   - ca-chain.pem: CA证书链, 可以验证此中间CA的CA证书和此中间CA签发的所有证书
>     - openssl verify -CAfile ca-chain.cert.pem ca-cert.pem
>   - fabric-ca-server-config.yaml: ca-server配置文件
>   - fabric-ca-server.db: ca-server数据库, 此数据库并不会与CA集群中其他CA节点的数据库同步, 事实上, 使用ca-server集群时, 应该在fabric-ca-server-config.yaml配置文件中配置使用单独节点上的postgresql或mysql数据库, 所有CA节点应该统一读写此远程数据库, 本地sqlite3数据库应该不存在
>   - msp/keystore/xxx_sk: 中间CA的私钥

### 说明

* 关于CA服务器集群下的CA数据库和HA Proxy负载均衡以及将Fabric-CA集成到Fabric网络的内容, 待续...

------

## 结语

完