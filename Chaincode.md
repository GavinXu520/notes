# 链码

本文将搭建一个简便的链码(chaincode)开发环境, 以操作者和开发者两种角度介绍链码相关知识.

---

## 目录

<!-- TOC -->

- [链码](#链码)
    - [目录](#目录)
    - [基本知识](#基本知识)
    - [环境](#环境)
        - [软件安装](#软件安装)
            - [依赖包](#依赖包)
            - [Go](#go)
            - [Fabric](#fabric)
            - [Fabric-Samples](#fabric-samples)
            - [Docker](#docker)
        - [资源获取](#资源获取)
        - [Fabric配置](#fabric配置)
    - [开发](#开发)
        - [chaincode interface](#chaincode-interface)
        - [chaincode stub interface](#chaincode-stub-interface)
        - [示例代码](#示例代码)
    - [操作](#操作)
        - [打包(package)](#打包package)
        - [签名(signpackage)](#签名signpackage)
        - [安装(install)](#安装install)
        - [实例化(instantiate)](#实例化instantiate)
        - [升级(upgrade)](#升级upgrade)
        - [查询(list)](#查询list)
        - [启停](#启停)
    - [其他](#其他)
        - [系统chaincode](#系统chaincode)
        - [docker脚本](#docker脚本)

<!-- /TOC -->

---

## 基本知识

- chaincode是一段由Go语言(当前还支持Java和nodejs)编写, 并能实现预定义接口的程序
- chaincode程序通常处理Fabric网络中相应channel的成员一致认可的业务逻辑, 因此Fabric中的chaincode即是区块链技术中智能合约概念的实现.
- chaincode运行在受保护的Docker容器中, 与背书节点的运行相互隔离, 只有当背书节点上发生交易请求时, 其对应的链码的Docker容器才会启动(启动后不会关闭)
- chaincode是系统与区块链(账本)进行交互的唯一途径, 业务交易通过为业务开发的应用链码读写区块链账本, Fabric网络的系统服务通过系统链码管理链码和账本
- 只有安装并实例化了chaincode的peer节点才能成为背书节点, 无法背书的peer节点只能进行交易的验证和记账
- 一个channel上可以有多个chaincode, 每个chaincode都有各自的摘要信息(digest), 同一channel上各个chaincode执行业务逻辑时创建的账本数据和世界状态数据之间是相互隔离的, 即一段chaincode不能直接访问其他chaincode创建的数据(对于couchdb实现的状态数据库, 不同链码产生的数据被存储在不同的库上), 但是同一channel上的各个chaincode之间是可以相互调用的, 因此可以间接实现数据互通
- 当前最新版本(v1.1.0)的chaincode不支持start与stop, 但支持upgrade, upgrade操作会导致旧版本的chaincode失效. 想要stop或者remove节点上的chaincode, 需要手动删除/var/hyperledger/production(默认)目录下相应的链码文件

---

## 环境

对于chaincode的开发与测试而言, 搭建完整的Fabric网络是没有必要的, 因此我们可以直接利用hyperledger提供的相关Docker容器, 搭建一个最小化的基于Docker的Fabric网络. 本开发环境的Fabric网络仅包含一个orderer节点和一个peer节点(这两个是必要节点), 为了方便链码安装和测试, 还启用了一个cli节点以命令行方式与Fabric API进行交互(当然也可以选择使用Fabric SDK). 此外, 本环境中状态数据库可选goleveldb和couchdb, 如果使用couchdb, 则还需要启用一个couchdb容器(当然也可以选择在peer节点单独配置一个couchdb服务). 这里我们通过修改fabric-sample提供的basic-network来实现这一需求.

> 以下内容均针对应用chiancode, 系统chiancode会单独说明
> 虽然hyperledger官方在fabric-sample中提供了一个chaincode-docker-devmode用于chaincode的开发和测试, 但是个人感觉并不方便

### 软件安装

#### 依赖包

用于编译相关源码.

```bash
yum install snappy-devel zlib-devel bzip2-devel libtool libtool-ltdl-devel -y
```

#### Go

版本要求 v1.9+, 官网下载, 用于下载.

```bash
wget https://studygolang.com/dl/golang/go1.10.linux-amd64.tar.gz
tar -zxvf go1.10.linux-amd64.tar.gz -C /usr/local/share/    #解压
mkdir -p /root/go         #创建go本地运行目录
vim /etc/profile          #创建go环境变量, 重要!
    ......
    #Go
    export GOROOT=/usr/local/share/go
    export GOPATH=/root/go
    export PATH=$GOROOT/bin:$GOPATH/bin:$PATH
source /etc/profile
```

#### Fabric

版本要求 v1.1.0,  用于链码开发.

```bash
go get -u github.com/hyperledger/fabric
cd $GOPATH/src/github.com/hyperledger/fabric
git tag                  #查看可用的fabric版本
git checkout v1.1.0      #切换为使用最新的v1.1.0版,重要!!!
```

#### Fabric-Samples

版本要求 v1.1.0, 用于编译生成Fabric服务环境.

```bash
go get -u github.com/hyperledger/fabric-samples
#也可以git clone:
#mkdir -p $GOPATH/src/github.com/hyperledger
#cd $GOPATH/src/github.com/hyperledger
#git clone https://github.com/hyperledger/fabric-samples.git
cd $GOPATH/src/github.com/hyperledger/fabric-samples
git tag                  #查看可用的fabric-samples版本
git checkout v1.1.0      #切换为使用最新的v1.1.0版,重要!!!
cd $GOPATH/src/github.com/hyperledger/fabric-samples/scripts
wget https://github.com/hyperledger/fabric/blob/release-1.1/scripts/bootstrap.sh  #fabric-samples的scripts目录中自带的fabric-preload.sh太旧, 需要使用fabric v1.1.0源码中的bootstrap.sh脚本拉取所需的二进制文件和Docker镜像
```

#### Docker

版本要求 v17.00+, yum安装, 用于源码编译和链码运行.

```bash
yum-config-manager --add-repo https://download.daocloud.io/docker/linux/centos/docker-ce.repo       #添加repo
yum install docker-ce -y                                                                            #安装docker
systemctl enable docker
systemctl start docker                                                                              #启动docker并设为开机运行
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://64ff7d79.m.daocloud.io      #加速docker镜像下载
curl -L https://get.daocloud.io/docker/compose/releases/download/1.12.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose                                                              #安装docker-compose, 用于从baseimage和yaml配置文件生成docker镜像并运行
```

### 资源获取

使用上述工具拉取所需的二进制文件,和Docker镜像.

```bash
cd $GOPATH/src/github.com/hyperledger/fabric-samples/scripts
bash bootstrip.sh 1.1.0    #拉取二进制文件和Docker镜像, 二进制文件会保存在当前bin目录下, Docker镜像会保存在Docker默认的镜像目录下, 使用docker images命令查看
cp bin/* $GOPATH/bin/      #将二进制文件复制到$PATH的任意目录下
```

### Fabric配置

1. 从fabric-samples中提取basic-network.

```bash
cp -r $GOPATH/src/github.com/hyperledger/fabric-samples/basic-network ~/
```

2. 生成创世区块, 通道文件和相关证书密钥.

```bash
cd ~/basic-network
bash generate.sh               #生成的创世块和通道文件会保存在config目录中, 证书和密钥会保存在crypto-config目录中
```

3. 创建相关目录

```bash
cd ~/basic-network
mkdir -p chaincode/example     #开发的链码保存在此位置, cli节点会挂载此目录用于获取链码
mkdir -p data                  #peer节点的数据目录(包含链码, 区块链账本和状态数据库等)会挂载到此目录, 方便查看
```

3. 配置容器
docker-compose基于baseimage和docker-compose.yml文件的配置创建容器, 这里配置orderer, peer, cli, couchdb四个容器.

```bash
cd ~/basic-network
vim docker-compose.yml
##################################################################
#
# Copyright IBM Corp All Rights Reserved
#
# SPDX-License-Identifier: Apache-2.0
#
version: '2'

networks:
  basic:

services:

  orderer.example.com:                         #orderer节点容器配置
    container_name: orderer.example.com
    image: hyperledger/fabric-orderer          #基于此baseimage创建容器
    environment:                               #指定相关环境变量
      - ORDERER_GENERAL_LOGLEVEL=debug         #debug模式日志输出,方便调试
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_GENESISMETHOD=file     #使用创世区块文件
      - ORDERER_GENERAL_GENESISFILE=/etc/hyperledger/configtx/genesis.block
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP
      - ORDERER_GENERAL_LOCALMSPDIR=/etc/hyperledger/msp/orderer/msp
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/orderer
    command: orderer                              #容器启动后执行命令
    ports:
      - 7050:7050                                 #本地和容器端口映射
    volumes:
        - ./config/:/etc/hyperledger/configtx     #将创世区块和通道文件挂载进容器中
        - ./crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/:/etc/hyperledger/msp/orderer
        - ./crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/:/etc/hyperledger/msp/peerOrg1                                    #将证书密钥挂载进容器中
    networks:
      - basic

  peer0.org1.example.com:
    container_name: peer0.org1.example.com
    image: hyperledger/fabric-peer
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_PEER_ID=peer0.org1.example.com
      - CORE_LOGGING_PEER=debug
      - CORE_CHAINCODE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_LOCALMSPID=Org1MSP
      - CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/peer/
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
      # # the following setting starts chaincode containers on the same
      # # bridge network as the peers
      # # https://docs.docker.com/compose/networking/
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=${COMPOSE_PROJECT_NAME}_basic
      - CORE_LEDGER_STATE_STATEDATABASE=CouchDB     #配置使用goleveldb或CouchDB
      - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb:5984
      #不设置CouchDB用户密码表示任何登录者均是数据库管理员
      - CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME=   
      - CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD=
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: peer node start
    # command: peer node start --peer-chaincodedev=true
    ports:
      - 7051:7051
      - 7053:7053
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp:/etc/hyperledger/msp/peer
        - ./crypto-config/peerOrganizations/org1.example.com/users:/etc/hyperledger/msp/users
        - ./config:/etc/hyperledger/configtx
        - ./data:/var/hyperledger/production      #将默认的Fabric数据目录映射到宿主机
    depends_on:
      - orderer.example.com
      - couchdb
    networks:
      - basic

  couchdb:
    container_name: couchdb
    image: hyperledger/fabric-couchdb
    environment:
    #此处用户和密码的配置需要与peer节点中的相应配置项一致, 否则peer节点无法向其写入数据
      - COUCHDB_USER=
      - COUCHDB_PASSWORD=
    ports:
      - 5984:5984      # 通过http://<couchdb_ip>:5984/_utils访问CouchDB
    networks:
      - basic

  cli:
    container_name: cli
    image: hyperledger/fabric-tools
    tty: true
    environment:
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_ID=cli
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
      - CORE_CHAINCODE_KEEPALIVE=10
      - CAF=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
      - FABRIC_CFG_PATH=/etc/hyperledger/fabric
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: /bin/bash
    volumes:
        - /var/run/:/host/var/run/
        - ./chaincode:/opt/gopath/src/github.com/    #将宿主机的chaincode目录映射到cli节点默认的链码查找目录下
        - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
    networks:
        - basic
    #depends_on:
    #  - orderer.example.com
    #  - peer0.org1.example.com
    #  - couchdb
##################################################################
```

4. 编辑start.sh
此脚本用于启动容器并启动Fabric网络.

```bash
#!/bin/bash
#
# Copyright IBM Corp All Rights Reserved
#
# SPDX-License-Identifier: Apache-2.0
#
# Exit on first error, print all commands.
set -ev

# don't rewrite paths for Windows Git Bash users
export MSYS_NO_PATHCONV=1
export COMPOSE_PROJECT_NAME=$(basename $PWD)      #docker-compose命令使用此环境变量或者--project-name或-p参数指定项目名称, 项目名称和docker-compose.yml中指定的networks组合生成docker网络名

docker-compose -f docker-compose.yml down         #确保上次产生的Docker容器被关闭

if [[ $(docker ps -a | wc -l) > 1 ]];             #会删除所有容器(如果存在其他用途的容器, 请删除此部分)
then
    docker rm -f $(docker ps -a -q)
fi

sudo rm -rf config/*                              #清空上次运行的痕迹
sudo rm -rf crypto-config/*
sudo rm -rf data/*
sudo rm -rf chaincode/hyperledger

./generate.sh                                     #生成创世块, 通道文件, 证书密钥等

# docker-compose -f docker-compose.yml up -d ca.example.com orderer.example.com peer0.org1.example.com couchdb
docker-compose -f docker-compose.yml up -d orderer.example.com peer0.org1.example.com couchdb cli                #启动四个容器

# wait for Hyperledger Fabric to start
# incase of errors when running later commands, issue export FABRIC_START_TIMEOUT=<larger number>
export FABRIC_START_TIMEOUT=10
#echo ${FABRIC_START_TIMEOUT}
sleep ${FABRIC_START_TIMEOUT}                     #等待容器启动完成

# Create the channel
docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@org1.example.com/msp" peer0.org1.example.com peer channel create -o orderer.example.com:7050 -c mychannel -f /etc/hyperledger/configtx/channel.tx              #创建通道
# Join peer0.org1.example.com to the channel.
docker exec -e "CORE_PEER_LOCALMSPID=Org1MSP" -e "CORE_PEER_MSPCONFIGPATH=/etc/hyperledger/msp/users/Admin@org1.example.com/msp" peer0.org1.example.com peer channel join -b mychannel.block    #加入通道
```

5. 编辑stop.sh
此脚本用于清除上一次的运行痕迹(此功能已经加入start.sh脚本中)

```bash
#!/bin/bash
#
# Copyright IBM Corp All Rights Reserved
#
# SPDX-License-Identifier: Apache-2.0
#
# Exit on first error, print all commands.
set -ev
export COMPOSE_PROJECT_NAME=$(basename $PWD)


# Shut down the Docker containers that might be currently running.
docker-compose -f docker-compose.yml stop
if [[ $(docker ps -a | wc -l) > 1 ]];
then
    docker rm -f $(docker ps -a -q)
fi
sudo rm -rf config/*
sudo rm -rf crypto-config/*
sudo rm -rf data/*
sudo rm -rf chaincode/hyperledger
```

6. 编写chain.sh
此脚本用于将chaincode目录下的链码自动安装并实例化到peer节点上(执行前请确保~/basic-network/chaincode/example目录中存在链码文件).

```bash
#!/bin/bash

version=$(cat /proc/sys/kernel/random/uuid| cksum | cut -f1 -d" ") #生成随机的链码版本号(如果每次使用相同的版本号会出错, 原因未明)
docker exec cli cd $FABRIC_CFG_PATH
docker exec cli peer chaincode package -n mycc -p github.com/example -v $version mycc.pak                                 #打包链码
docker exec cli peer chaincode install mycc.pak   #安装链码
docker exec cli peer chaincode list -C mychannel --installed   #查看已安装链码
docker exec cli peer chaincode instantiate -o orderer.example.com:7050 -C mychannel -n mycc -v $version -c '{"Args":["init"]}' -P "OR('Org1MSP.member')"                            #实例化链码
sleep 5                                           #等待链码实例化完成
docker exec cli peer chaincode list -C mychannel --instantiated   #查看已实例化链码
```

7. 测试链码
依次执行start.sh和chain.sh脚本即可启动Fabric网络并完成链码的实例化. 然后便可以测试链码的可用性了. 修改链码后, 执行stop.sh并再次以此执行start.sh和chain.sh即可测试新的链码(执行stop.sh可省略). 可以编写一个简单的自动测试的脚本如下(依链码具体功能而定):

```bash
#!/bin/bash
case $1 in
    add)
        docker exec cli peer chaincode invoke -C mychannel -n mycc -c '{"Args":["Add","Customer","1","Tom","12.3"]}'
        docker exec cli peer chaincode invoke -C mychannel -n mycc -c '{"Args":["Add","Customer","2","Luna","23.4"]}'
        docker exec cli peer chaincode invoke -C mychannel -n mycc -c '{"Args":["Add","Customer","3","Nick","34.5"]}'
        docker exec cli peer chaincode invoke -C mychannel -n mycc -c '{"Args":["Add","Customer","4","Nick","45.6"]}'
        docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["Traverse","Customer"]}'
        ;;
        # echo "============================================================================================================"
        # $leep 5
    query)
        docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["Query","Customer","",""]}'
        ;;
    update)
        docker exec cli peer chaincode invoke -C mychannel -n mycc -c '{"Args":["Update","Customer","Id","1","Deposit","34.5"]}'
        sleep 5
        docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["Query","Customer","Name","Tom"]}'
        ;;
        # echo "============================================================================================================"
    delete)
        docker exec cli peer chaincode invoke -C mychannel -n mycc -c '{"Args":["Delete","Customer","Name","Nick"]}'
        sleep 5
        docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["Query","Customer","Name","Nick"]}'
        ;;
        # echo "============================================================================================================"
    transact)
        docker exec cli peer chaincode invoke -C mychannel -n mycc -c '{"Args":["Transact","Customer","1","2","Deposit","5"]}'
        sleep 5
        docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["Traverse"]}'
        ;;
        # echo "============================================================================================================"
    traverse)
        docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["Traverse"]}'
        ;;
    histquery)
        docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["HistQuery","Customer","",""]}'
        ;;
    *)
        docker exec cli peer chaincode query -C mychannel -n mycc -c '{"Args":["Test","Welcome"]}'
esac
```

---

## 开发

chaincode是一段实现了预定义接口的代码,  目前支持使用包括go, node, java在内的多种语言编写(官方主推go chaincode).  本文仅讨论go语言实现方式.

### chaincode interface

chaincode interface即是chaincode必须实现的预定义接口,  该接口包含两个函数: Init和Invoke.
- Init: chaincode在接收到instantiate(实例化)或upgrade(升级)请求时会调用Init方法进行必要的初始化操作.
- Invoke: chaincode在接收到query(查询)或invoke(调用)请求时会调用Invoke方法执行具体的交易请求. 
> query和invoke的区别(对chaincode的操作都被称为交易transaction):
> - query交易根据请求读取状态数据库并返回响应,  query交易是不会被写入区块链中的, 即便query交易调用的方法包含写操作.
> - invoke交易根据请求读写状态数据库和区块链账本并返回响应, invoke交易一定会被写入区块链中, 即便invoke交易调用的方法仅包含读操作.

Init 和Invoke函数的参数都是一个实现了fabric的shim库中的ChaincodeStubInterface接口的ChaincodeStub类实例, 返回值都是fabric的peer库的Response类实例.
> Response类包含三个成员变量:
> - Status: 返回状态码
> - Message: 返回消息
> - Payload: 交易或响应对象的字节数组

chaincode的开发模板: 首先声明一个自定义类(结构体)并实现Init和Invoke函数, 然后在Init和Invoke中实现业务逻辑, 最后在main函数中实例化该类, 并将其注册到peer节点中, 监听交易请求.

### chaincode stub interface

chaincode stub interface(存根接口)定义了一系列与交易处理相关的函数, 包含交易参数的解析, 状态数据库和区块链账本的读写, chaincode的调用, 交易及其发起者信息的获取, 事件设置等.

> - func (stub *ChaincodeStub) GetArgs() [][]byte : 获取交易参数列表(二维字节数组形式)
> - func (stub *ChaincodeStub) GetArgs() []string: 获取交易参数列表(字符串形式)
> - func (stub *ChaincodeStub) GetArgsSlice() ([]byte, error) : 获取交易参数列表(字节数组切片形式)
> - func (stub *ChaincodeStub) GetFunctionAndParameters() (function string, params []string) : 获取交易函数和参数列表(即将第一个参数当做请求函数名)
> - func (stub *ChaincodeStub) GetTxID() string : 获取交易ID
> - func (stub *ChaincodeStub) GetTxTimestamp() (*timestamp.Timestamp, error) : 获取交易时间戳
> - func (stub *ChaincodeStub) GetCreator() ([]byte, error) : 获取交易发起者的证书
> - func (stub *ChaincodeStub) GetBinding() ([]byte, error) : 获取交易的绑定(交易ID和交易对象/资产的所有者的签名, 绑定可以实现资产所有者委托其他人对自己的资产进行一次交易)
> - func (stub *ChaincodeStub) GetTransient() (map[string][]byte, error) : 获取交易请求的临时数据(如加解密的认证和密钥材料等)
> - func (stub *ChaincodeStub) GetSignedProposal() (*pb.SignedProposal, error) : 获取已签名的交易请求的解密后的完整请求对象
> - func (stub *ChaincodeStub) GetDecorations() map[string][]byte : 获取交易请求的额外修饰信息
> - func (stub *ChaincodeStub) GetState(key string) ([]byte, error): 获取指定key在状态数据库中的值
> - func (stub *ChaincodeStub) GetHistoryForKey(key string) (HistoryQueryIteratorInterface, error): 获取指定key的历史交易记录
> - func (stub *ChaincodeStub) GetStateByRange(startKey, endKey string) (StateQueryIteratorInterface, error): 获取指定key范围的数据
> - func (stub *ChaincodeStub) GetQueryResult(query string) (StateQueryIteratorInterface, error) : 以富查询方式获取查询结果集(不支持leveldb, 因为leveldb中键值均是byte, 无法支持基于值的富查询)
> - func (stub *ChaincodeStub) CreateCompositeKey(objectType string, attributes []string) (string, error) : 创建由多个属性组成的复合key
> - func (stub *ChaincodeStub) SplitCompositeKey(compositeKey string) (string, []string, error) : 将复合key拆分
> - func (stub *ChaincodeStub) GetStateByPartialCompositeKey(objectType string, attributes []string) (StateQueryIteratorInterface, error) : 获取指定复合key的查询结果集
> - func (stub *ChaincodeStub) PutState(key string, value []byte) error : 写入数据
> - func (stub *ChaincodeStub) DelState(key string) error : 删除指定key的数据(并非真的删除数据,只是将数据状态设置为已删除)
> - func (stub *ChaincodeStub) InvokeChaincode(chaincodeName string, args [][]byte, channel string) pb.Response : 链码调用
> - func (stub *ChaincodeStub) SetEvent(name string, payload []byte) error : 链码为交易设置事件, 交易完成后将事件分发给所有事件监听者, 交易发起端可以根据收到的请求响应获取交易消息

### 示例代码

以一段示例代码演示简单的chaincode业务逻辑.

```Go
//mycc.go
package main

import (
    "encoding/json"
    "errors"
    "fmt"
    "reflect"
    "strconv"
    "strings"

    "github.com/hyperledger/fabric/core/chaincode/shim"
    "github.com/hyperledger/fabric/protos/peer"
)

type MC struct {
}

type KV struct {
    Key   string
    Value []byte
}
type Customer struct {
    Id      int
    Name    string
    Deposit float64
}

func (t *MC) Init(stub shim.ChaincodeStubInterface) peer.Response {
    return shim.Success([]byte("OK"))
}

func (t *MC) Invoke(stub shim.ChaincodeStubInterface) peer.Response {
    function, args := stub.GetFunctionAndParameters()
    return t.Parse(stub, function, args)
}

func (t *KV) Display() string {
    return t.Key + ":" + string(t.Value)
}

//根据给定参数获取kv结果集
func (t *MC) FetchKVs(stub shim.ChaincodeStubInterface, stype, skey, svalue string) ([]KV, error) {
    var kvs []KV
    if skey == "Id" {
        var key string
        if stype != "" {
            key = stype + ":" + svalue
        } else {
            key = svalue
        }
        item, err := stub.GetState(key)
        if err != nil {
            return kvs, errors.New("None")
        }
        kvs = append(kvs, KV{key, item})
    } else {
        if stype == "" && skey != "" {
            return kvs, errors.New("None")
        } else {
            startKey := "0"
            endKey := "ZZZZZZZZZZZZZZZ"
            if stype != "" {
                startKey = stype + ":" + startKey
                endKey = stype + ":" + endKey
            }

            keysIter, err := stub.GetStateByRange(startKey, endKey)
            if err != nil {
                return kvs, err
            }
            defer keysIter.Close()

            for keysIter.HasNext() {
                resp, iterErr := keysIter.Next()
                if iterErr != nil {
                    return kvs, iterErr
                }
                if stype != "" && skey != "" {
                    switch stype {
                    case "Customer":
                        var customer Customer
                        err = json.Unmarshal(resp.Value, &customer)
                        if err != nil {
                            return kvs, err
                        }

                        item := reflect.ValueOf(&customer).Elem()
                        for i := 0; i < item.NumField(); i++ {
                            if item.Type().Field(i).Name == skey {
                                itype := item.Field(i).Type().Name()
                                var tvalue string
                                switch itype {
                                case "int":
                                    tvalue = strconv.Itoa(int(item.Field(i).Int()))
                                case "string":
                                    tvalue = item.Field(i).String()
                                case "float64":
                                    tvalue = strconv.FormatFloat(item.Field(i).Float(), 'f', -1, 64)
                                }
                                if svalue == tvalue {
                                    kvs = append(kvs, KV{resp.Key, resp.Value})
                                }
                                break
                            }
                        }
                    default:
                        break
                    }
                } else {
                    kvs = append(kvs, KV{resp.Key, resp.Value})
                }
            }
        }
    }
    return kvs, nil
}

//根据请求参数中指定的函数将请求交由对应的函数处理
func (t *MC) Parse(stub shim.ChaincodeStubInterface, function string, args []string) peer.Response {
    switch function {
    case "Query":
        return t.QueryAgent(stub, args)
    case "Add":
        return t.AddAgent(stub, args)
    case "Update":
        return t.UpdateAgent(stub, args)
    case "Delete":
        return t.DeleteAgent(stub, args)
    case "Transact":
        return t.TransactAgent(stub, args)
    case "Traverse":
        return t.Traverse(stub, args)
    case "HistQuery":
        return t.HistQuery(stub, args)
    case "Test":
        return t.Test(stub, args)
    default:
        return shim.Error("Parse error")
    }
}

func (t *MC) QueryAgent(stub shim.ChaincodeStubInterface, args []string) peer.Response {
    if args == nil || len(args) != 3 {
        return shim.Error("QueryAgent error")
    }
    switch args[0] {
    case "Customer":
        return t.QueryCustomer(stub, args[1], args[2])
    default:
        return shim.Error("QueryAgent error")
    }
}

func (t *MC) AddAgent(stub shim.ChaincodeStubInterface, args []string) peer.Response {
    if args == nil || len(args) != 4 {
        return shim.Error("AddAgent error")
    }
    switch args[0] {
    case "Customer":
        return t.AddCustomer(stub, args[1], args[2], args[3])
    default:
        return shim.Error("AddAgent error")
    }
}

func (t *MC) UpdateAgent(stub shim.ChaincodeStubInterface, args []string) peer.Response {
    if args == nil || len(args) != 5 {
        return shim.Error("UpdateAgent error")
    }
    switch args[0] {
    case "Customer":
        return t.UpdateCustomer(stub, args[1], args[2], args[3], args[4])
    default:
        return shim.Error("UpdateAgent error")
    }
}

func (t *MC) DeleteAgent(stub shim.ChaincodeStubInterface, args []string) peer.Response {
    if args == nil || len(args) != 3 {
        return shim.Error("DeleteAgent error")
    }
    switch args[0] {
    case "Customer":
        return t.DeleteCustomer(stub, args[1], args[2])
    default:
        return shim.Error("DeleteAgent error")
    }
}

func (t *MC) TransactAgent(stub shim.ChaincodeStubInterface, args []string) peer.Response {
    if args == nil || len(args) != 5 {
        return shim.Error("TransactionAgent error")
    }
    switch args[0] {
    case "Customer":
        return t.TransactCustomer(stub, args[1], args[2], args[3], args[4])
    default:
        return shim.Error("TransactionAgent error")
    }
}

//查询
func (t *MC) QueryCustomer(stub shim.ChaincodeStubInterface, ikey, ivalue string) peer.Response {
    var result []string
    kvs, err := t.FetchKVs(stub, "Customer", ikey, ivalue)
    if err != nil {
        return shim.Error(err.Error())
    }

    for _, kv := range kvs {
        result = append(result, kv.Display())
    }
    return shim.Success([]byte("Queryed Customer:" + strings.Join(result, ",")))
}

//添加
func (t *MC) AddCustomer(stub shim.ChaincodeStubInterface, iid, iname, ideposit string) peer.Response {
    id, err := strconv.Atoi(iid)
    if err != nil {
        return shim.Error(err.Error())
    }

    name := iname

    deposit, err := strconv.ParseFloat(ideposit, 64)
    if err != nil {
        return shim.Error(err.Error())
    }
    deposit = float64(deposit)

    customer := Customer{id, name, deposit}

    key := "Customer:" + strconv.Itoa(customer.Id)
    value, err := json.Marshal(customer)
    if err != nil {
        return shim.Error(err.Error())
    }

    err = stub.PutState(key, value)
    if err != nil {
        return shim.Error(err.Error())
    }

    return shim.Success([]byte("Added Customer:" + key + ":" + string(value)))
}

//更新
func (t *MC) UpdateCustomer(stub shim.ChaincodeStubInterface, ikey, ivalue, ukey, uvalue string) peer.Response {
    var skvs []string
    var fkvs []string
    kvs, err := t.FetchKVs(stub, "Customer", ikey, ivalue)
    if err != nil {
        return shim.Error(err.Error())
    }

    for _, kv := range kvs {
        var customer Customer
        err = json.Unmarshal(kv.Value, &customer)
        if err != nil {
            return shim.Error(err.Error())
        }

        item := reflect.ValueOf(&customer).Elem()
        for i := 0; i < item.NumField(); i++ {
            if item.Type().Field(i).Name == ukey {
                itype := item.Field(i).Type().Name()
                switch itype {
                case "int":
                    nvalue, err := strconv.Atoi(uvalue)
                    if err != nil {
                        return shim.Error(err.Error())
                    }
                    item.FieldByName(ukey).SetInt(int64(nvalue))
                case "string":
                    nvalue := uvalue
                    item.FieldByName(ukey).SetString(nvalue)
                case "float64":
                    nvalue, err := strconv.ParseFloat(uvalue, 64)
                    if err != nil {
                        return shim.Error(err.Error())
                    }
                    item.FieldByName(ukey).SetFloat(nvalue)
                }
                break
            }
        }

        value, err := json.Marshal(customer)
        if err != nil {
            return shim.Error(err.Error())
        }

        err = stub.PutState(kv.Key, value)
        if err != nil {
            fkvs = append(fkvs, kv.Key+":"+string(value))
        } else {
            skvs = append(skvs, kv.Key+":"+string(value))
        }
    }
    return shim.Success([]byte("Updated Customer Success:" + strings.Join(skvs, ",") + "\nUpdated Customer Fail:" + strings.Join(fkvs, ",")))
}

//删除
func (t *MC) DeleteCustomer(stub shim.ChaincodeStubInterface, ikey, ivalue string) peer.Response {
    var skvs []string
    var fkvs []string
    kvs, err := t.FetchKVs(stub, "Customer", ikey, ivalue)
    if err != nil {
        return shim.Error(err.Error())
    }

    for _, kv := range kvs {
        err := stub.DelState(kv.Key)
        if err != nil {
            fkvs = append(fkvs, kv.Display())
        } else {
            skvs = append(skvs, kv.Display())
        }
    }
    return shim.Success([]byte("Deleted Customer Success:" + strings.Join(skvs, ",") + "\nDeleted Customer Fail:" + strings.Join(fkvs, ",")))
}

//实例间交易
func (t *MC) TransactCustomer(stub shim.ChaincodeStubInterface, ivalue, jvalue, tkey, tvalue string) peer.Response {
    var icustomer Customer
    var jcustomer Customer
    ikvs, err := t.FetchKVs(stub, "Customer", "Id", ivalue)
    if err != nil {
        return shim.Error(err.Error())
    }

    for _, kv := range ikvs {
        err = json.Unmarshal(kv.Value, &icustomer)
        if err != nil {
            return shim.Error(err.Error())
        }
    }

    jkvs, err := t.FetchKVs(stub, "Customer", "Id", jvalue)
    if err != nil {
        return shim.Error(err.Error())
    }

    for _, kv := range jkvs {
        err = json.Unmarshal(kv.Value, &jcustomer)
        if err != nil {
            return shim.Error(err.Error())
        }
    }

    //attrs := reflect.TypeOf(icustomer)
    iitem := reflect.ValueOf(&icustomer).Elem()
    jitem := reflect.ValueOf(&jcustomer).Elem()
    for i := 0; i < iitem.NumField(); i++ {
        if iitem.Type().Field(i).Name == tkey {
            ttype := iitem.Field(i).Type().Name()
            switch ttype {
            case "int":
                ntvalue, err := strconv.Atoi(tvalue)
                if err != nil {
                    return shim.Error("TransactionCustomer error")
                }
                if int(iitem.FieldByName(tkey).Int()) < ntvalue {
                    return shim.Error("TransactionCustomer not sufficient funds")
                }
                nivalue := int(iitem.FieldByName(tkey).Int()) - ntvalue
                njvalue := int(jitem.FieldByName(tkey).Int()) + ntvalue
                iitem.FieldByName(tkey).SetInt(int64(nivalue))
                jitem.FieldByName(tkey).SetInt(int64(njvalue))
            case "string":
                return shim.Error("TransactionCustomer error")
            case "float64":
                ntvalue, err := strconv.ParseFloat(tvalue, 64)
                if err != nil {
                    return shim.Error("TransactionCustomer error")
                }
                if iitem.FieldByName(tkey).Float() < ntvalue {
                    return shim.Error("TransactionCustomer not sufficient funds")
                }
                nivalue := iitem.FieldByName(tkey).Float() - ntvalue
                njvalue := jitem.FieldByName(tkey).Float() + ntvalue
                iitem.FieldByName(tkey).SetFloat(nivalue)
                jitem.FieldByName(tkey).SetFloat(njvalue)
            }
            break
        }
    }

    bivalue, ierr := json.Marshal(icustomer)
    bjvalue, jerr := json.Marshal(jcustomer)
    if ierr != nil || jerr != nil {
        return shim.Error(ierr.Error() + jerr.Error())
    }

    err = stub.PutState(ikvs[len(ikvs)-1].Key, bivalue)
    if err != nil {
        return shim.Error(err.Error())
    }
    err = stub.PutState(jkvs[len(jkvs)-1].Key, bjvalue)
    if err != nil {
        return shim.Error(err.Error())
    }

    return shim.Success([]byte("Transacted Customer Success"))
}

//遍历
func (t *MC) Traverse(stub shim.ChaincodeStubInterface, args []string) peer.Response {
    var result []string
    rtype := ""
    if args != nil && len(args) > 0 && strings.TrimSpace(args[0]) != "" {
        rtype = args[0]
    }
    kvs, err := t.FetchKVs(stub, rtype, "", "")
    if err != nil {
        return shim.Error(err.Error())
    }

    for _, kv := range kvs {
        result = append(result, kv.Display())
    }

    return shim.Success([]byte("Traverse " + rtype + ":" + strings.Join(result, ",")))

}

//历史记录查询
func (t *MC) HistQuery(stub shim.ChaincodeStubInterface, args []string) peer.Response {
    var result []string

    kvs, err := t.FetchKVs(stub, args[0], args[1], args[2])
    if err != nil {
        return shim.Error(err.Error())
    }
    for _, kv := range kvs {
        histIter, err := stub.GetHistoryForKey(kv.Key)
        if err != nil {
            return shim.Error(err.Error())
            // return shim.Error(fmt.Sprintf("keys operation failed. Error accessing state: %s", err))
        }
        defer histIter.Close()

        for histIter.HasNext() {
            resp, iterErr := histIter.Next()
            if iterErr != nil {
                return shim.Error(err.Error())
                // return shim.Error(fmt.Sprintf("keys operation failed. Error accessing state: %s", err))
            }
            item, _ := json.Marshal(resp)
            result = append(result, string(item))
        }
    }

    return shim.Success([]byte("Traverse " + strings.Join(result, ",")))

}

func (t *MC) Test(stub shim.ChaincodeStubInterface, args []string) peer.Response {
    tid := stub.GetTxID()
    ttime, _ := stub.GetTxTimestamp()
    cc,_ := stub.GetCreator()
    bind, _ := stub.GetBinding()
    res := string(tid) + ":" + string(ttime.String()) + ":" + string(cc) + ":" + string(bind)
    return shim.Success([]byte("Echo: " + strings.Join(args, ",") + "=>" + res))
}

func main() {
    if err := shim.Start(new(MC)); err != nil {
        fmt.Printf("Error starting MC chaincode: %s", err)
    }
}
```

---

## 操作

可以通过Hyperledger Fabric API与Fabric网络中的各种节点(peer, orderer, ca等)进行交互, 对chaincode的操作其实就是利用API与peer节点的一种交互类型. 使用API的途径有两种: SDK和CLI, SDK是编程开发的交互方式, 当前版本支持node, java和go语言(其中go语言还没有正式文档支持); CLI是直接通过命令行进行交互的方式, 本文使用CLI方式演示chaincode相关操作.

chaincode编写完成后, 需要安装到peer节点上, 然后进行实例化, 才能被Fabric API调用. 如有需要, 还可对chaincode进行升级. 

### 打包(package)

对chaincode进行打包的时候可以在生成的package中包含chaincode的名称, 版本, 实例化策略和所有者签名等内容.

```bash
peer chaincode package -n mycc -p github.com/example -v 1.0 -s -S -i "AND('Org1.admin')" mycc.pak
```

> 说明:
> -n 选项(必选)表示chaincode名称
> -v 选项(必选)表示chaincode版本, 名称与版本唯一标识一个chaincode
> -p 选项(必选)表示chaincode程序位置
> -s 选项(可选)表示创建的包可以被多个所有者签名
> -S 选项(可选)表示该节点对包进行签名(签名使用的id和私钥通过节点的core.yaml配置文件或者环境变量中指定的LOCALMSPID和LOCALMSPDIR获取), 生成签名的包后, 其他节点对包签名使用peer chaincode signpackage命令. 如果生成包的时候没有使用-S进行签名, 则其他节点也不允许再对其签名了
> -i 选项(可选)可以指定chaincode的实例化策略,  实例化策略类似背书策略, 限定谁可以实例化该chaincode包(限定的是用户, 而不是节点), 默认情况下仅允许peer节点的MSP管理员实例化chaincode

### 签名(signpackage)

可以在打包时对生成的chaincode包签名, 也可以对已有的chaincode包进行签名.

```bash
peer chaincode signpackage mycc.pak mycc.a.pak
```

### 安装(install)

只有安装了chaincode的peer节点才能对该chaincode的请求进行背书. 可以安装chaincode包, 也可以直接指定chaincode源代码所在目录进行安装. 安装chaincode时, cli会为chaincode创建SignedChaincodeDeploymentSpec, 然后将其发送到peer节点, peer节点会调用LSCC系统链码上的Install方法, 完成chaincode安装.

```bash
peer chaincode install -n mycc -v 1.0 -p github.com/example   #源代码安装
peer chaincode install mycc.pak                            #包安装
```

### 实例化(instantiate)

chaincode实例化时peer节点会验证其实例化策略, 只有实例化后的chaincode才能处理交易请求. 每个peer节点上的chaincode实例化后都会生成一个对应的chaincode容器, chaincode在容器中处理交易. 

```bash
peer chaincode instantiate -o orderer.example.com:7050 -C mychannel -n mycc -v 1.0 -c '{"Args":["init"]}' -P "OR('Org1MSP.member')"
```

> 说明:
> -C 选项(必选)表示将chaincode实例化到哪个channel上
> -n 选项(必选)表示chaincode名称
> -v 选项(必选)表示chaincode版本
> -c 选项(必选)表示实例化时的初始化参数
> -P 选项(必选)表示背书策略
> -o 选项(可选)表示共识/排序节点
>  chaincode安装和实例化时, 可以使用--tls和--cafile选项指定使用加密方式进行数据传输

### 升级(upgrade)

通过安装一个与已有chaincode具有相同名称不同版本号的新chaincode, 然后实例化新chaincode, 可以对chaincode进行升级, chaincode升级后, 旧版本将不再处于已实例化状态(即不可用), 而仅是已安装状态, 新版本chaincode仍然可以处理旧版本chaincode生成的数据. 升级操作不会影响其他channel上对旧版本chaincode的使用. 实例化新chaincode的时候, peer节点验证的是旧版本chaincode的实例化策略, 确保只有当前实例化策略指定的成员才能升级该chaincode.

```bash
#假设已有一个名为mycc, 版本号为1.0的chaincode已经实例化
peer chaincode install -n mycc -v 2.0 -p github.com/example
peer chaincode upgrade -o orderer.example.com:7050 -C mychannel -n mycc -v 2.0 -c '{"Args":["init"]}' -P "OR('Org1MSP.member')"
```

### 查询(list)

可以对已安装和已实例化的chaincode进行查询.

```bash
peer chaincode list -C mychannel --installed     #查询已安装chaincode
peer chaincode list -C mychannel --instantiated  #查询已实例化chaincode
```

### 启停

Fabric v1.1.0还不支持chaincode的启动与停止指令, 可以通过移除chaincode容器并删除peer节点fabric数据目录下的chaincode包来停用chaincode.

---

## 其他

### 系统chaincode

系统chaincode与普通chaincode的编程模型相同，只不过它运行于peer节点内而非一个隔离的容器中。因此，系统chaincode在节点内构建且不遵循上文描述的chaincode生命周期。特别地，安装，实例化，升级这三项操作不适用于系统chaincode。

系统chaincode的目的是削减peer节点和chaincode之间的gRPC通讯成本，并兼顾管理的灵活性。例如：一个系统chaincode只能通过peer节点的二进制文件升级。同时，系统chaincode只能以一组编译好的特定的参数进行注册，且不具有背书策略相关功能。

系统chaincode在Hyperledger Fabric中用于实现一些系统行为，故它们可以被系统开发者适当替换或更改。

> 系统chaincode列表：
> - LSCC：生命周期系统chaincode处理上述生命周期相关的功能
> - CSCC：配置系统chaincode处理peer侧channel的配置
> - QSCC：查询系统chaincode提供账本查询API，比如获取区块及交易等
> - ESCC：背书系统chaincode通过对交易响应进行签名来处理背书过程
> - VSCC：验证系统chaincode处理交易的验证，包括检查背书策略以及多版本并发控制

替换或更改这些系统chaincode一定要万分小心，尤其是LSCC, ESCC 和 VSCC，因为它们处于主交易执行路径中。值得注意的是，VSCC在一个区块被提交到账本之前进行验证，故所有channel中的peer节点得出相同的验证结果以避免账本分叉（不确定因素）就很重要了。所以当VSCC被更改或替换时就要特别小心了。

### docker脚本

一个方便查看容器信息的脚本

```bash
#!/bin/bash
#Description:   shell下格式化输出为表格样式
#               使用时首先需要调用set_title对表格初始化
#               追加表格数据可使用append_cell和append_line，append_cell不会自动换行，换行必须要使用append_line
#               append_line参数是可选的，并且会自动对之前的append_cell换行
#               使用output_table可输出表格
#               暂不支持修改/插入/删除数据
#               可使用. format_table.sh 或者source format_table.sh来引入改脚本的函数
#               "(*)"会自动着色为红色字体
# +----+------+---------------+
# |ID  |Name  |Creation time  |
# +----+------+---------------+
# |1   |TF    |2017-01-01     |
# |2   |      |2017-01-02(*)  |
# |3   |SF    |               |
# |3   |SF    |(*)            |
# |4   |TS    |               |
# |5   |      |               |
# +----+------+---------------+


sep="#"
function append_cell(){
    #对表格追加单元格
    #append_cell col0 "col 1" ""
    #append_cell col3
    local i
    for i in "$@"
    do
        line+="|$i${sep}"
    done
}
function check_line(){
if [ -n "$line" ] 
then
    c_c=$(echo $line|tr -cd "${sep}"|wc -c)
    difference=$((${column_count}-${c_c}))
    if [ $difference -gt 0 ]
    then
        line+=$(seq -s " " $difference|sed -r s/[0-9]\+/\|${sep}/g|sed -r  s/${sep}\ /${sep}/g)
    fi
    content+="${line}|\n"
fi

}
function append_line(){
    check_line
    line=""
    local i
    for i in "$@"
    do
        line+="|$i${sep}"
    done
    check_line
    line=""
}
function segmentation(){
    local seg=""
    local i
    for i in $(seq $column_count)
    do 
        seg+="+${sep}"
    done
    seg+="${sep}+\n"
    echo $seg
}
function set_title(){
    #表格标头，以空格分割，包含空格的字符串用引号，如
    #set_title Column_0 "Column 1" "" Column3
    [ -n "$title" ] && echo "Warring:表头已经定义过,重写表头和内容"
    column_count=0
    title=""
    local i
    for i in "$@"
    do
        title+="|${i}${sep}"
        let column_count++
    done
    title+="|\n"
    seg=`segmentation`
    title="${seg}${title}${seg}"
    content=""
}
function output_table(){
    if [ ! -n "${title}" ] 
    then
        echo "未设置表头，退出" && return 1
    fi
    append_line
    table="${title}${content}$(segmentation)"
    echo -e $table|column -s "${sep}" -t|awk '{if($0 ~ /^+/){gsub(" ","-",$0);print $0}else{gsub("\\(\\*\\)","\033[31m(*)\033[0m",$0);print $0}}'

}

if [ "$SHLVL" -eq "2" ]
then
    set_title "容器名" "主机名" "网络模式" "网络地址"
    append_line 1 "TF" "2017-01-01"
    append_cell 2 "" "2017-01-02(*)"
    append_line 
    append_cell 3 "SF"
    append_line 
    append_cell 3 "SF" "(*)"
    append_line 4 "TS"
    append_cell 5
    output_table
fi

IFS=$'\n'
containers=$(docker inspect -f '{{.Name}}✡{{.Config.Hostname}}✡{{.Config.Image}}✡{{.Path}} {{.Args}}✡{{.HostConfig.NetworkMode}}✡{{range .NetworkSettings.Networks}}{{.IPAddress}}/{{.IPPrefixLen}}{{end}}✡{{.NetworkSettings.Ports}}' $(docker ps -aq))

set_title "容器名" "主机名" "镜像" "命令" "网络模式" "网络地址" "端口映射"
for container in  $containers
do
    if [[ $# == 0 ]]
    then
        item=$(echo $container) 
    else
        item=$(echo $container | grep $1)
    fi

    if [[ $item != "" ]]
    then
        cname=$(echo $item | cut -d '✡' -f1)
        cname=${cname//"/"/""}
        hname=$(echo $item | cut -d '✡' -f2)
        image=$(echo $item | cut -d '✡' -f3)
        cmmnd=$(echo $item | cut -d '✡' -f4)
        cmmnd=${cmmnd//"["/""}
        cmmnd=${cmmnd//"]"/""}
        nmode=$(echo $item | cut -d '✡' -f5)
        ipadr=$(echo $item | cut -d '✡' -f6)
        ports=$(echo $item | cut -d '✡' -f7)
        ports=${ports//"0.0.0.0"/""}
        ports=${ports//"map"/""}
        ports=${ports//"["/""}
        ports=${ports//"]"/""}
        ports=${ports//"{"/""}
        ports=${ports//"}"/""}
        ports=${ports//": "/":"}
        ports=${ports//" "/"|"}

        append_line $cname $hname $image $cmmnd $nmode $ipadr $ports
    fi
done
output_table
```
