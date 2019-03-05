# SDK

FISCO BCOS提供了Java语言SDK用以访问节点，查询节点状态，改变节点设置和发送交易等功能。

该版本（2.0）的技术文档只适用Java SDK V2.0及以上版本(与FISCO BCOS V2.0及以上版本适配)，Java SDK V1.3.x版本的技术文档请查看[Java SDK V1.3.x版本技术文档](https://fisco-bcos-documentation.readthedocs.io/zh_CN/latest/docs/web3sdk/config_web3sdk.html)。[Java SDK Github](https://github.com/FISCO-BCOS/web3sdk)提供访问FISCO BCOS节点的Java API。

- 提供调用FISCO BCOS JSON-RPC的Java API   
- [链上信使协议](../manual/amop_protocol.html)，为联盟链提供安全高效的消息信道。
- 支持使用国密算法发交易
- 支持预编译（Precompiled）合约管理区块链

## 环境要求

```eval_rst
.. important::

    - java版本
     要求 `JDK8或以上 <https://openjdk.java.net/>`_，推荐使用OpenJDK11
     > CentOS的yum仓库的OpenJDK由于缺少JCE(Java Cryptography Extension)，导致Java SDK无法正常连接区块链节点，在使用CentOS操作系统时，推荐从OpenJDK网站自行下载。`OpenJDK11下载地址 <https://jdk.java.net/11/>`_ `安装指南 <https://openjdk.java.net/install/index.html>`_ 
    - FISCO BCOS区块链环境搭建
     参考 `FISCO BCOS搭链教程 <../installation.html>`_
    - 网络连通性
     检查Java SDK连接的FISCO BCOS节点channel_listen_port是否能telnet通，若telnet不通，需要检查网络连通性和安全策略。

```

## Java应用引入SDK

   通过gradle或maven引入SDK到java应用

   gradle:
```bash
compile ('org.fisco-bcos:web3sdk:2.0.2')
```
   maven:
```bash
<dependency>
    <groupId>org.fisco-bcos</groupId>
    <artifactId>web3sdk</artifactId>
              <version>2.0.2</version>
</dependency>
```
由于引入了以太坊的solidity编译器相关jar包，需要在Java应用的gradle配置文件build.gradle中添加以太坊的远程仓库

```bash
repositories {
        mavenCentral()
        maven { url "https://dl.bintray.com/ethereum/maven/" }
    }
```
## 配置SDK

### FISCO BCOS节点证书配置
FISCO-BCOS作为联盟链，其SDK连接区块链节点需要通过证书(ca.crt、node.crt)和私钥(node.key)进行双向认证。

- **通过[建链脚本](../manual/build_chain.md)搭建的节点证书配置：** 需要将节点所在目录nodes/${ip}/sdk下的ca.crt、node.crt和node.key文件拷贝到项目的资源目录。
- **通过[企业工具](../enterprise/index.html)搭建的区块节点证书配置：** 企业工具的demo命令生成的证书和私钥与建链脚本相同。如果使用企业工具的build和expand命令，则需要自己生成证书和私钥，或者使用企业工具的--sdkca命令(具体参考企业工具的[证书生成相关命令](../enterprise/manual/cert.md))生成证书和私钥，将生成sdk目录下的ca.crt、node.crt和node.key文件拷贝到项目的资源目录。

### 配置文件设置
Java应用的配置文件需要做相关配置。值得关注的是，FISCO-BCOS2.0支持[多群组功能](../design/architecture/group.md)，SDK需要配置群组的节点信息。将以Spring项目和Spring Boot项目为例，提供配置指引。

### Spring项目配置
提供Spring项目中关于applicationContext.xml的配置如下图所示，其中红框标记的内容根据区块链节点配置做相应修改。

![](../../images/sdk/sdk_xml.png)

applicationContext.xml配置项详细说明:
- encryptType: 国密算法开关(默认为0)                              
  - 0: 不使用国密算法发交易                              
  - 1: 使用国密算法发交易(开启国密功能，需要连接的区块链节点是国密节点，搭建国密版FISCO BCOS区块链[参考这里](../manual/guomi.md))
- groupChannelConnectionsConfig: 
  - 配置待连接的群组，可以配置一个或多个群组，每个群组需要配置群组ID 
  - 每个群组可以配置一个或多个节点，设置群组节点的配置文件**config.ini**中`[rpc]`部分的`listen_ip`和`channel_listen_port`。
- channelService: 通过指定群组ID配置SDK实际连接的群组，指定的群组ID是groupChannelConnectionsConfig配置中的群组ID。SDK会与群组中配置的节点均建立连接，然后随机选择一个节点发送请求。

### Spring Boot项目配置
提供Spring Boot项目中关于application.yml的配置如下图所示，其中红框标记的内容根据区块链节点配置做相应修改。

![](../../images/sdk/sdk_yml.png)

application.yml配置项与applicationContext.xml配置项相对应，详细介绍参考applicationContext.xml配置说明。

## 使用SDK 

### Spring项目开发指引
#### 调用SDK的API(参考[SDK API列表](./api.md))设置或查询相关的区块链数据。
1) 调用SDK Web3j的API：需要加载配置文件，SDK与区块链节点建立连接。获取web3j对象，根据Web3j对象调用相关API。示例代码如下：
```java
    //读取配置文件，SDK与区块链节点建立连接
    ApplicationContext context = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
    Service service = context.getBean(Service.class);
    service.run(); 
    ChannelEthereumService channelEthereumService = new ChannelEthereumService();
    channelEthereumService.setChannelService(service);
    //获取Web3j对象
    Web3j web3j = Web3j.build(channelEthereumService, service.getGroupId());
    //通过Web3j对象调用API接口getBlockNumber
    BigInteger blockNumber = web3j.getBlockNumber().send().getBlockNumber();
    System.out.println(blockNumber);
```
2) 调用SDK Precompiled的API：需要加载配置文件，SDK与区块链节点建立连接。获取SDK Precompiled Service对象，调用相关的API。示例代码如下：
```java
    //读取配置文件，SDK与区块链节点建立连接，获取Web3j对象
    ApplicationContext context = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
    Service service = context.getBean(Service.class);
    service.run(); 
    ChannelEthereumService channelEthereumService = new ChannelEthereumService();
    channelEthereumService.setChannelService(service);
    Web3j web3j = Web3j.build(channelEthereumService, service.getGroupId());
    //填入用户私钥，用于交易签名
    Credentials credentials = Credentials.create("b83261efa42895c38c6c2364ca878f43e77f3cddbc922bf57d0d48070f79feb6"); 
    //获取SystemConfigService对象
    SystemConfigSerivce systemConfigSerivce = new SystemConfigSerivce(web3j, credentials);
    //通过SystemConfigService对象调用API接口setValueByKey
    String result = systemConfigSerivce.setValueByKey("tx_count_limit", "2000");
    //通过Web3j对象调用API接口getSystemConfigByKey
    String value = web3j.getSystemConfigByKey("tx_count_limit").send().getSystemConfigByKey();
    System.out.println(value);
```

#### 通过SDK部署并调用合约
##### 准备Java合约文件：Solidity合约文件转换为Java合约文件
使用SDK把Solidity合约文件转成相应Java合约文件。合约转换工作在SDK源码目录下进行，因此需要获取SDK源码，然后进行合约转换。

```bash 
# 获取sdk源码
git clone https://github.com/FISCO-BCOS/web3sdk
cd web3sdk
git checkout release-2.0.1
```
合约转换步骤：
- 把编写的Solidity合约文件拷贝到src/test/resources/contract下，确保合约名和文件名保持一致。
- 在SDK根目录下执行转换合约命令:
    ```
    bash sol2java
    ```
  生成的Java合约文件在src/test/java/org/fisco/bcos/temp目录，生成的abi和bin文件在src/test/resources/solidity目录。

**温馨提示：SDK默认支持的Solidity版本是0.4.25，如果编译0.5以上版本合约，需要做如下配置**
- 修改build.gradle配置文件，注释0.4.25版编译器jar包，使用0.5.2版编译器jar包，修改后的配置如下：
  ```
   // compile files('lib/solcJ-all-0.4.25-gm.jar')
      compile 'org.ethereum:solcJ-all:0.5.2'
   // compile 'org.ethereum:solcJ-all:0.4.25'
  ```
- 删除src/test/resources/contract目录下版本为0.4.x的Solidity合约文件，将版本为0.5.x的Solidity合约文件拷贝到src/test/resources/contract目录下。
- 在SDK根目录下执行转换合约命令:
    ```
    bash sol2java
    ```
  生成的Java合约文件在src/test/java/org/fisco/bcos/temp目录，生成的abi和bin文件在src/test/resources/solidity目录。


##### 部署并调用合约
SDK的核心功能是部署/加载合约，然后调用合约相关接口，实现相关业务功能。部署合约调用Java合约类的deploy方法，获取合约对象。通过合约对象可以调用getContractAddress方法获取部署合约的地址以及调用该合约的其他方法实现业务功能。如果合约已部署，则通过部署的合约地址可以调用load方法加载合约对象，然后调用该合约的相关方法。
```bash
    //读取配置文件，sdk与区块链节点建立连接，获取web3j对象
    ApplicationContext context = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
    Service service = context.getBean(Service.class);
    service.run(); 
    ChannelEthereumService channelEthereumService = new ChannelEthereumService();
    channelEthereumService.setChannelService(service);
    channelEthereumService.setTimeout(10000);
    Web3j web3j = Web3j.build(channelEthereumService, service.getGroupId());
    //准备部署和调用合约的参数
    BigInteger gasPrice = new BigInteger("300000000");
    BigInteger gasLimit = new BigInteger("300000000");
    Credentials credentials = Credentials.create("b83261efa42895c38c6c2364ca878f43e77f3cddbc922bf57d0d48070f79feb6");
    //部署合约 
    YourSmartContract contract = YourSmartContract.deploy(web3j, credentials, new StaticGasProvider(gasPrice, gasLimit)).send();
    //根据合约地址加载合约
    //YourSmartContract contract = YourSmartContract.load(address, web3j, credentials, new StaticGasProvider(gasPrice, gasLimit)); 
    //调用合约方法发送交易
    TransactionReceipt transactionReceipt = contract.someMethod(<param1>, ...).send(); 
    //查询合约方法查询该合约的数据状态
    Type result = contract.someMethod(<param1>, ...).send(); 
```
### Spring Boot项目开发指引
提供[spring-boot-starter](https://github.com/FISCO-BCOS/spring-boot-starter)示例项目供参考。Spring Boot项目开发与Spring项目开发类似，其主要区别在于配置文件方式的差异。该示例项目提供丰富的测试案例，具体描述参考示例项目的README文档。

### SDK国密功能使用
- 前置条件：FISCO BCOS区块链采用国密算法，搭建国密版的FISCO BCOS区块链请参考[国密使用手册](../manual/guomi.md)。
- 启用国密功能：application.xml/application.yml配置文件中将encryptType属性设置为1。

国密版SDK调用API的方式与普通版SDK调用API的方式相同，其差异在于国密版SDK需要生成国密版的Java合约文件。SDK源码的lib目录下已有国密版的编译器jar包，用于将Solidity合约文件转为国密版的Java合约文件。因此，使用国密版SDK只需要修改SDK源码根目录下的build.gradle文件，注释掉普通版编译器jar包，引入国密编译器jar包。
  ```
      compile files('lib/solcJ-all-0.4.25-gm.jar')
   // compile 'org.ethereum:solcJ-all:0.4.25'
   // compile 'org.ethereum:solcJ-all:0.5.2'
  ```
Solidity合约文件转换为国密版Java合约文件的步骤、部署和调用国密版合约的方法均与普通版SDK相同。

## Java SDK API

Java SDK API主要分为Web3j API和Precompiled Service API。其中Web3j API可以查询区块链相关的状态，发送和查询交易信息；Precompiled Service API可以管理区块链相关配置以及实现特定功能。

### Web3j API
Web3j API是由web3j对象调用的FISCO BCOS的RPC API，其API名称与RPC API相同，参考[RPC API文档](../api.md)。调用Web3j API示例参考[SDK配置与使用](./config.md)。

### Precompiled Service API
预编译合约是FISCO BCOS底层通过C++实现的一种高效智能合约。SDK已提供预编译合约对应的Java接口，控制台通过调用这些Java接口实现了相关的操作命令，体验控制台，参考[控制台手册](../manual/console.md)。SDK提供Precompiled对应的Service类，分别是分布式控制权限相关的AuthorityService，[CNS](../design/features/CNS_contract_name_service.md)相关的CnsService，系统属性配置相关的SystemConfigService和节点类型配置相关ConsensusService。

设置和移除接口返回json字符串，包含code和msg字段，当无权限操作时，其code定义-1，msg定义为“non-authorized”。当成功操作时，其code为1(影响1条记录)，msg为“success”。其他返回码如下：

|错误码|消息内容|
|:----|:---|
|0|success|
|50000|permission denied|
|51000|table name and address exist|
|51001|table name and address does not exist|
|51100|invalid nodeId|
|51101|the last sealer cannot be removed|
|51102|table name and address does not exist|
|51103|the node is not in group peers|
|51104|the node is already in sealer list|
|51105|the node is already in observer list|
|51200|contract name and version exist|
|51201|version exceeds maximum(40) length|
|51300|invalid configuration key|
调用Precompiled Service API示例参考[Java SDK配置与使用](./config.md)。

#### AuthorityService
SDK提供对[分布式控制权限](../manual/permission_control.md)的支持，AuthorityService可以配置权限信息，其API如下：
- **public String grantUserTableManager(String tableName, String address)：** 根据用户表名和外部账号地址设置权限信息。
- **public String revokeUserTableManager(String tableName, String address)：** 根据用户表名和外部账号地址去除权限信息。
- **public List\<AuthorityInfo\> listUserTableManager(String tableName)：** 根据用户表名查询设置的权限记录列表(每条记录包含外部账号地址和生效块高)。
- **public String grantDeployAndCreateManager(String address)：** 增加外部账号地址的部署合约和创建用户表权限。
- **public String revokeDeployAndCreateManager(String address)：** 移除外部账号地址的部署合约和创建用户表权限。
- **public List\<AuthorityInfo\> listDeployAndCreateManager()：** 查询拥有部署合约和创建用户表权限的权限记录列表。
- **public String grantPermissionManager(String address)：** 增加外部账号地址的管理权限的权限。
- **public String revokePermissionManager(String address)：** 移除外部账号地址的管理权限的权限。
- **public List\<AuthorityInfo\> listPermissionManager()：** 查询拥有管理权限的权限记录列表。
- **public String grantNodeManager(String address)：** 增加外部账号地址的节点管理权限。
- **public String revokeNodeManager(String address)：** 移除外部账号地址的节点管理权限。
- **public List\<AuthorityInfo\> listNodeManager()：** 查询拥有节点管理的权限记录列表。
- **public String grantCNSManager(String address)：** 增加外部账号地址的使用CNS权限。
- **public String revokeCNSManager(String address)：** 移除外部账号地址的使用CNS权限。
- **public List\<AuthorityInfo\> listCNSManager()：** 查询拥有使用CNS的权限记录列表。
- **public String addSysConfig(String address)：** 增加外部账号地址的系统参数管理权限。
- **public String removeSysConfig(String address)：** 移除外部账号地址的系统参数管理权限。
- **public List\<AuthorityInfo\> querySysConfig()：** 查询拥有系统参数管理的权限记录列表。

#### CnsService
SDK提供对[CNS](../design/features/CNS_contract_name_service.md)的支持。CnsService可以配置CNS信息，其API如下：
- **String registerCns(String name, String version, String address, String abi)：** 根据合约名、合约版本号、合约地址和合约abi注册CNS信息。
- **String getAddressByContractNameAndVersion(String contractNameAndVersion)：** 根据合约名和合约版本号(合约名和合约版本号用英文冒号连接)查询合约地址。
- **List\<CnsInfo\> queryCnsByName(String name)：** 根据合约名查询CNS信息。
- **List\<CnsInfo\> queryCnsByNameAndVersion(String name, String version)：** 根据合约名和合约版本号查询CNS信息。

#### SystemConfigSerivce
SDK提供对系统配置的支持。SystemConfigSerivce可以配置系统属性值（目前支持tx_count_limit和tx_gas_limit属性的设置），其API如下：
- **String setValueByKey(String key, String value)：** 根据键设置对应的值（查询键对应的值，参考Web3j API中的getSystemConfigByKey接口）。

#### ConsensusService 
SDK提供对[节点类型](../design/security_control/node_management.md)配置的支持。ConsensusService可以设置节点类型，其API如下：
- **String addSealer(String nodeId)：** 根据节点NodeID设置对应节点为共识节点。
- **String addObserver(String nodeId)：** 根据节点NodeID设置对应节点为观察节点。
- **String removeNode(String nodeId)：** 根据节点NodeID设置对应节点为游离节点。