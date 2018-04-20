# Fabric SDK for Node

<!-- TOC -->

- [Fabric SDK for Node](#fabric-sdk-for-node)
    - [Overview](#overview)
    - [Pluggable API](#pluggable-api)
        - [CryptoSuite](#cryptosuite)
            - [CryptoSuite概述](#cryptosuite概述)
            - [CryptoSuite方法](#cryptosuite方法)
                - [setCryptoKeyStore](#setcryptokeystore)
                - [generateKey](#generatekey)
                - [importKey](#importkey)
                - [getKey](#getkey)
                - [encrypt](#encrypt)
                - [decrypt](#decrypt)
                - [deriveKey](#derivekey)
                - [hash](#hash)
                - [sign](#sign)
                - [verify](#verify)
        - [Key](#key)
            - [Key概述](#key概述)
            - [Key方法](#key方法)
                - [getPublicKey](#getpublickey)
                - [getSKI](#getski)
                - [generateCSR](#generatecsr)
                - [isPrivate](#isprivate)
                - [isSymmetric](#issymmetric)
                - [toBytes](#tobytes)
        - [KeyValueStore](#keyvaluestore)
            - [KeyValueStore概述](#keyvaluestore概述)
            - [KeyValueStore方法](#keyvaluestore方法)
                - [setValue](#setvalue)
                - [getValue](#getvalue)
    - [Fabri-ca-client](#fabri-ca-client)
    - [Fabric-client](#fabric-client)

<!-- /TOC -->

## Overview

Hyperledger Fabric是一个企业级的支持授权管理的区块链网络操作系统, Hyperledger Fabric SDK for Node.js是一个用于Node.js Javascript运行时环境, 提供强大与Hyperledger Fabric区块链网络交互的API的应用程序开发套件, 本SDK提供的API支持的主要功能包括:

- 创建通道
- peer节点加入通道
- peer节点安装链码
- 在通道中实例化链码
- 调用链码进行交易
- 查询交易账本和区块

Hyperledger Fabric SDK for Node.js主要由三个顶层模块构成:

- 可插拔api: 可插拔API模块为应用程序开发者提供可选的供SDK使用的关键接口的实现, 每一个接口都有内建的默认实现
- fabric-ca-client: ca-client模块提供与可选的提供成员管理服务的组件fabric-ca进行交互的API
- fabric-client: client模块提供与Fabric区块链网络的核心组件(即peer节点, orderer节点, 事件流等)进行交互的API

---

## Pluggable API

可插拔API主要包括三个抽象类:

- CryptoSuite: 加密套件, 它提供了SDK执行数字签名, 加/解密, 安全散列等操作的一系列加密算法. 一个完整的加密套件应当包含对非对称密钥算法(如ECDSA, RSA等), 对称密钥算法(如AES等)和安全散列算法(如SHA2, SHA3等)的支持, Fabric SDK提供的是基于AES + ECDSA + SHA2/3的加密套件实现. 当然也可以使用"crypto-suite-software"实现另外的加密套件, 作为模块引入Fabric SDK中.
- Key: 密钥类, 密钥可以是对称或者非对称的, 对于非对称密钥, 密钥类的对象可以是公钥也可以是私钥. 非对称密钥算法的数学特性, 根据私钥是可以直接推导出公钥的.
- KeyValueStore: 键值存储类, 用于存储Key-Value的仓库, Channel类将使用此存储库保存诸如认证用户的私钥, 证书等敏感信息.

### CryptoSuite

#### CryptoSuite概述

CryptoSuite抽象类有两个具体的实现类: CryptoSuite_ECDSA_AES(基于软件的实现)和CryptoSuite_PKCS11(基于硬件的实现,即HSM). 两种实现方式均使用ECDSA算法, 默认采用的实现方式是CryptoSuite_ECDSA_AES算法, 通过fabric-client模块的newCryptoSuite()方法创建CryptoSuite_ECDSA_AES(keysize, hash)类的对象(ICryptoSuite), 加密套件的默认实现类, ECDSA算法默认使用的密钥长度(keysize)和hash算法(hash), 默认的加密key-value文件存储路径和tls加密数据传输(grpc-ssl)支持的对称加密算法等均在fabric-client/config/default.json文件中定义. CryptoSuite_ECDSA_AES类的方法的具体实现均在fabric-client/lib/impl/CryptoSuite_ECDSA_AES.js文件中. CryptoSuite_PKCS11类的方法的具体实现均在fabric-client/lib/impl/bccsp_pkcs11.js文件中.

#### CryptoSuite方法

> 只介绍CryptoSuite_ECDSA_AES类的方法.

##### setCryptoKeyStore

- **格式**: ICryptoSuite.setCryptoKeyStore(cryptoKeyStore: CryptoKeyStore)
- **描述**: CryptoSuite_ECDSA_AES对象(ICryptoSuite)的setCryptoKeyStore方法会为密钥文件设置存储位置.
- **参数**: 参数cryptoKeyStore为CryptoKeyStore类型(该类型由fabric-client对象的newCryptoKeyStore(obj: {path: string;})方法创建), 指定密钥文件存储位置.
- **返回**: 无返回值.
- **示例**:

```javascript
var Client = require('fabric-client');

var store_path = './key-store';
var crypto_store = Client.newCryptoKeyStore({path: store_path});          //创建CryptoKeyStore对象
crypto_suite.setCryptoKeyStore(crypto_store);                             //指定CryptoSuite对象(ICryptoSuite)的密钥存储位置
```

##### generateKey

- **格式**: ICryptoSuite.generateKey(opts: KeyOpts): Promise<ICryptoKey>
- **描述**: CryptoSuite_ECDSA_AES对象(ICryptoSuite)的generateKey方法会生成一个非对称密钥对(key pair).
- **参数**: 参数opts为KeyOpts对象(实现自定义接口interface KeyOpts{ephemeral: boolean;}), ephemeral指定了生成的密钥是否需要持久化到硬盘上, ephemeral=true表示生成临时的密钥, ephemeral=false表示密钥将作为PEM文件保存到key store中(此时需要先指定CryptoKeyStore, 否则报错), 持久化到key store中的密钥可以通过getKey()方法获取.
- **返回**: 返回值是一个CryptoKey对象(ICryptoKey)的Promise, Promise是一个带有then()方法的对象, then()方法的第一个参数是Resolved状态(成功)的回调函数, 第二个参数(可选)是Rejected状态(失败)的回调函数. 返回的ICryptoKey是密钥对中的私钥, 可以调用其getPublicKey()方法获得公钥.
- **示例**:

```javascript
var Client = require('fabric-client');

var crypto_suite = Client.newCryptoSuite();                     //创建CryptoSuite_ECDSA_AES对象
crypto_suite.generateKey({ephemeral: true}).then((sk) => {      //lambda匿名函数
    console.log(sk.toBytes());                                  //输出私钥
});
/*
var store_path = './key-store';
var crypto_store = Client.newCryptoKeyStore({path: store_path});
crypto_suite.setCryptoKeyStore(crypto_store);                    //指定CryptoKeyStore后, 密钥便可以持久化到硬盘
crypto_suite.generateKey().then((sk) => {                       //等同于{ephemeral: false}, 私钥将以PEM文件的形式存储在硬盘上
    console.log(sk.toBytes());                                  //输出私钥
    console.log(sk.getPublicKey().toBytes());
});
*/
```

> **[注]**: ICryptoSuite的generateKey方法使用的是KEYUTIL的generateKeypair函数:<static> {Array} KEYUTIL.generateKeypair(alg, keylenOrCurve), 说明如下:
> - alg参数表示算法, 支持两种:RSA和ECDSA(椭圆曲线数字签名算法,Elliptic CurveDigital Signature Algorithm)
> - keylenOrCurve参数表示安全强度, 安全强度在RSA算法上以密钥长度度量(一般1024bit), 在ECDSA算法上以弧度度量(支持的弧度名:secp256r1, secp256k1, secp384r1, 分别对应256和384bit长度的密钥)
> - RSA算法的破解难度是亚指数级, ECDSA算法的破解难度是指数级. RSA和ECDSA生成的密钥同等长度下, ECDSA密钥安全强度更高, 因此ECDSA算法更有利于提高交易速度,减少存储空间
> - Fabric的CryptoSuite仅支持ECDSA算法, Fabric默认使用CryptoSuite_ECDSA_AES, keysize为256(即secp256r1), hash为SHA2

##### importKey

- **格式**: ICryptoSuite.importKey(pem: string, opts: KeyOpts): ICryptoKey | Promise<ICryptoKey>
- **描述**: 将密钥的原始字符串形式导入为CryptoKey对象.
- **参数**: 参数pem为密钥的原始字符串,即pem文件的全部内容(不包括换行符); 参数opts为KeyOpts对象(同generateKey()方法), 指定产生的CryptoKey对象是否需要持久化到硬盘.
- **返回**: 如果opts.ephemeral为false或者opts参数为空(默认), 则返回值为CryptoKey对象ICryptoKey的Promise, 否则为CryptoKey对象.
- **示例**:

```javascript
var Client = require('fabric-client');

var crypto_suite = Client.newCryptoSuite();                      //创建CryptoSuite_ECDSA_AES对象
var crypto_key = crypto_suite.importKey('-----BEGIN PRIVATE KEY-----MIGHAgEAMBMGByqGSM49AgEGCCqGSM49AwEHBG0wawIBAQQgnyP9C+/cTvtz3QHCnHNnh8r/tSZup5hHu0+GBe+R3eChRANCAARgBwxwFoXajY3Z55RKQOiu5u8DLwD95jOa0UwnVTsOPVmJnObz2AkweDamaFlHGv4lt/XPgTyQqwWIYtnTFwTP-----END PRIVATE KEY-----', {ephemeral: true});                                              //注意pem字符串中没有换行
console.log(crypto_key.toBytes());                               //导入的k密钥产生CryptoKey对象与generateKey()方法生成的CryptoKey对象没有区别
console.log(crypto_key.getPublicKey().toBytes())                 //如果crypto_key是私钥, 则返回其对应的公钥, 如果crypto_key是公钥, 则返回其自身
```

##### getKey

- **格式**: ICryptoSuite.getKey(ski: string): Promise<ICryptoKey>
- **描述**: 根据SKI(主体密钥标识符, Subject Key Identifier)从KeyStore中获取私钥. SKI即是一个密钥集合中密钥的唯一标识符(或者说索引), 就是密钥持久化到KeyStore中的PEM文件的文件名前缀, 一个密钥对的公私钥拥有相同的SKI. CryptoKey对象的getSKI()方法可以获取密钥的SKI.
- **参数**: 参数ski为要获取的密钥的ski.
- **返回**: 返回值为拥有指定ski的私钥.
- **示例**:

```javascript
var Client = require('fabric-client');

var store_path = './key-store';
var crypto_store = Client.newCryptoKeyStore({path: store_path});   //创建CryptoKeyStore对象
crypto_suite.setCryptoKeyStore(crypto_store);                      //设置KeyStore

crypto_suite.generateKey({ephemeral: false}).then((sk) => {        //在KeyStore中生成私钥
    console.log(sk.isPrivate());
    var ski = sk.getSKI();                                         //获取私钥的ski
    crypto_suite.getKey(ski).then((tsk) => {                       //根据ski获取私钥
        console.log(tsk);
        console.log(tsk.toBytes() == sk.toBytes());                //二者相同
    });
});
```

##### encrypt

- **格式**: ICryptoSuite.encrypt(key: ICryptoKey, plainText: Array.<byte>, opts: Object): Array.<byte>
- **描述**: 使用公钥对文本进行加密.
- **参数**: 参数key为CryptoKey对象(公钥); 参数plainText为需要加密的字节数组; 参数opts指定加密算法.
- **返回**: 返回值为加密后的字节数组.
- **示例**: *本方法暂未实现*

##### decrypt

- **格式**: ICryptoSuite.decrypt(key: ICryptoKey, cipherText: Array.<byte>, opts: Object): Array.<byte>
- **描述**: 使用私钥对加密文本进行解密.
- **参数**: 参数key为CryptoKey对象(私钥); 参数cipherText为需要解密的字节数组; 参数opts指定解密算法.
- **返回**: 返回值为解密后的字节数组.
- **示例**: *本方法暂未实现*

> **[注]**: 非对称密钥的加解密操作只要加解密过程使用的密钥不同即可. 只是通常情况下都是用公钥加密, 私钥解密.

##### deriveKey

- **格式**: ICryptoSuite.deriveKey(key: ICryptoKey, opts: KeyOpts): ICryptoKey
- **描述**: 使用opts参数从源公钥推导出一个新的私钥, 这个操作可以用于根据交易证书(证书中包含公钥)推导私钥.
- **参数**: 参数key为CryptoKey对象(共钥); 参数opts为推导私钥所需的信息.
- **返回**: 返回值为推导出的私钥.
- **示例**: *本方法暂未实现*

##### hash

- **格式**: ICryptoSuite.hash(msg: string, opts: Object): string
- **描述**: 使用指定算法生成消息(字符串)的哈希值, 摘要(digest).
- **参数**: 参数msg为需要计算哈希的字符串; 参数opts为hash算法, 但目前无效, 会直接使用默认的sha2-256算法.
- **返回**: 返回值为消息的哈希值, 即摘要(digest).
- **示例**:

```javascript
var Client = require('fabric-client');

var crypto_suite = Client.newCryptoSuite();                      //创建CryptoSuite_ECDSA_AES对象
console.log(crypto_suite.hash("Hello", 'sha3'));                 //指定任何hash算法均无效, 仍会调用默认的sha2-256算法
```

##### sign

- **格式**: ICryptoSuite.sign(key: ICryptoKey, digest: Array, opts: Object): Array
- **描述**: 使用私钥对消息的摘要进行签名.
- **参数**: 参数key为进行签名的CryptoKey对象(私钥); 参数digest为消息的摘要, 参数opts为签名算法(此参数无效).
- **返回**: 返回值为消息的数字签名(十六进制数组).
- **示例**:

```javascript
var Client = require('fabric-client');

var msg = 'Hello';
var crypto_suite = Client.newCryptoSuite();
var digest = crypto_suite.hash(msg, 'sha2');
crypto_suite.generateKey({ephemeral: true}).then((sk) => {      //lambda匿名函数
    console.log(crypto_suite.sign(sk, digest, '').toString());  //使用内置方法进行签名, 无视opts参数
});
```

##### verify

- **格式**: ICryptoSuite.verify(key: ICryptoKey, signature: Array, msg: string): boolean
- **描述**: 使用公钥对消息摘要的数字签名进行验签.
- **参数**: 参数key为进行验签的CryptoKey对象(公钥); 参数signature为数字签名(十六进制数组), 参数msg为需要验签的消息.
- **返回**: 返回值为数字签名的真伪.
- **示例**:

```javascript
var Client = require('fabric-client');

var msg = 'Hello';
var crypto_suite = Client.newCryptoSuite();
var digest = crypto_suite.hash(msg, 'sha2');                    //生成消息摘要
crypto_suite.generateKey({ephemeral: true}).then((sk) => {      //生成私钥
    var pk = sk.getPublicKey();                                 //获取公钥
    var sign = crypto_suite.sign(sk, digest, '');               //生成签名
    console.log(crypto_suite.verify(pk, sign, msg));            //验签, 签名和验签的hash算法都是默认的sha2-256, 因此验签省略了hash算法
});
```

### Key

#### Key概述

与CryptoSuite抽象类一样, Key抽象类也有两个具体的实现类:ECDSA_KEY和PKCS11_ECDSA_KEY, 分别对应基于软件和硬件的实现. 前者的方法的具体实现均位于fabric-client/lib/impl/ecdsa/key.js文件中, 后者的方法的具体实现均位于fabric-client/lib/impl/ecdsa/pkcs11_key.js文件中.

#### Key方法

> 只介绍CryptoSuite_ECDSA_AES类的方法.

##### getPublicKey

- **格式**:  ICryptoKey.getPublicKey(): ICryptoKey
- **描述**: 根据私钥推导出公钥.
- **参数**: 无参数.
- **返回**: 返回值为私钥对应的公钥.
- **示例**:

```javascript
var Client = require('fabric-client');

var crypto_suite = Client.newCryptoSuite();                     //创建CryptoSuite_ECDSA_AES对象
crypto_suite.generateKey({ephemeral: true}).then((sk) => {      //生成私钥
    var pk = sk.getPublicKey();
    console.log(pk.toBytes());                                  //输出公钥
});
```

##### getSKI

- **格式**: ICryptoKey.getSKI(): string
- **描述**: 获取密钥(公私钥均可)的ski.
- **参数**: 无参数.
- **返回**: 返回值为密钥的ski(十六进制编码的字符串).
- **示例**:

```javascript
var Client = require('fabric-client');

var crypto_suite = Client.newCryptoSuite();                     //创建CryptoSuite_ECDSA_AES对象
crypto_suite.generateKey({ephemeral: true}).then((sk) => {      //生成私钥
    var pk = sk.getPublicKey();                                 //推导公钥
    console.log(sk.getSKI() + "<->" + pk.getSKI());             //查看公私钥的SKI
});
```

##### generateCSR

- **格式**: ICryptoKey.generateCSR(subjectDN: string): string
- **描述**: 通过私钥和LDAP格式的证书请求生成CSR(证书签名申请, 相当于将申请证书的必要材料进行了打包).
- **参数**: 参数subjectDN为LDAP格式的证书请求字符串.
- **返回**: 返回值为CSR字符串(用于向CA申请证书).
- **示例**:

```javascript
var Client = require('fabric-client');
var jsrsa = require('jsrsasign');
var KEYUTIL = jsrsa.KEYUTIL;

var crypto_suite = Client.newCryptoSuite();                     //创建CryptoSuite_ECDSA_AES对象
crypto_suite.generateKey({ephemeral: true}).then((sk) => {      //生成私钥
    var sdn = 'C=GB,ST=London,L=London,O=Global Security,OU=IT Department,CN=example.com';  //证书申请者的组织关系和服务器域名
    console.log(sk.generateCSR());                              //必须用私钥生成CSR, 生成的CSR中是包含公钥和申请者提交的相关信息的(可以通过在线网站对CSR字符串进行解析)
    console.log(KEYUTIL.getKeyFromCSRPEM(csr));                 //使用jsrsasign的getKeyFromCSRPEM方法从PEM格式的CSR字符串中得到十六进制的公钥
});
```

##### isPrivate

- **格式**: ICryptoKey.isPrivate(): boolean
- **描述**: 查看密钥是否是私钥.
- **参数**: 无参数.
- **返回**: 返回值为是否私钥的真假值.
- **示例**:

```javascript
var Client = require('fabric-client');

var crypto_suite = Client.newCryptoSuite();                     //创建CryptoSuite_ECDSA_AES对象
crypto_suite.generateKey({ephemeral: true}).then((sk) => {      //生成私钥
    var pk = sk.getPublicKey();                                 //推导公钥
    console.log(sk.isPrivate());
    console.log(pk.isPrivate());
});
```

##### isSymmetric

- **格式**: ICryptoKey.isSymmetric(): boolean
- **描述**: 查看密钥是否是对称密钥. 源代码中并无判断逻辑, 直接返回false.
- **参数**: 无参数.
- **返回**: 返回值为是否对称密钥的真假值.
- **示例**:

```javascript
var Client = require('fabric-client');

var crypto_suite = Client.newCryptoSuite();                     //创建CryptoSuite_ECDSA_AES对象
crypto_suite.generateKey({ephemeral: true}).then((sk) => {      //生成私钥
    var pk = sk.getPublicKey();                                 //推导公钥
    console.log(sk.isSymmetric());
    console.log(pk.isSymmetric());
});
```

##### toBytes

- **格式**: ICryptoKey.toBytes(): string
- **描述**: 将密钥对象转换为PEM格式的字符串(相当于打印密钥).
- **参数**: 无参数.
- **返回**: 返回值为密钥的PEM格式字符串.
- **示例**:

```javascript
var Client = require('fabric-client');

var crypto_suite = Client.newCryptoSuite();                     //创建CryptoSuite_ECDSA_AES对象
crypto_suite.generateKey({ephemeral: true}).then((sk) => {      //生成私钥
    var pk = sk.getPublicKey();                                 //推导公钥
    console.log(sk.toBytes());
    console.log(pk.toBytes());
});
```

### KeyValueStore

#### KeyValueStore概述

KeyValueStore抽象类也有两个具体实现类: FileKeyValueStore和CouchDBKeyValueStore, 前者基于文件, 后者基于CouchDB数据库, 其中FileKeyValueStore是默认的实现类, 通过fabric-client模块的newDefaultKeyValueStore{path: store_path}可以构建一个KeyValueStore对象. 通过Fabric-Client类的对象可以通过setStateStore(kvs: IKeyValueStore)为此对象应用此KeyValueStore对象, 此后Fabric-Client类的对象生成的相关文件均会保存在此存储库中. 此存储库对应于Fabirc账本中的状态数据库.

> 只介绍FileKeyValueStore类的方法.

#### KeyValueStore方法

##### setValue

- **格式**: IKeyValueStore.setValue(name: string, value: string): Promise<string>
- **描述**: 向存储库中存储键值对.
- **参数**: 参数name为键; 参数value为值.
- **返回**: 返回值为值的Promise. 另外会在存储库中生成名为键,内容为值的文件. 该文件中不会保存值的历史记录.
- **示例**:

```Javascript
var Client = require('fabric-client');
//var util = require('util');

var store_path = './key-store'
var client = new Client();

Client.newDefaultKeyValueStore({path: store_path}).then((store) => {
    client.setStateStore(store);
    store.setValue("name", "tom").then((value) => {
        console.log(value);
    });
});
```

##### getValue

- **格式**: IKeyValueStore.getValue(name: string): Promise<string>
- **描述**: 根据键从存储库中获取值.
- **参数**: 参数name为键.
- **返回**: 返回值为值的Promise.
- **示例**:

```Javascript
var Client = require('fabric-client');
//var util = require('util');

var store_path = './key-store'
var client = new Client();

Client.newDefaultKeyValueStore({path: store_path}).then((store) => {
    client.setStateStore(store);
    store.setValue("name", "nick");                 //重新设置值后相应文件中只会保存新的值
    store.getValue("name").then((value) => {
        console.log(value);
    });
});
```

---

## Fabri-ca-client

> 待续

---

## Fabric-client

> 待续