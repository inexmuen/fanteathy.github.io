---
title: 基于Pylon的Demo开发及发布相关
date: 2016-12-30 15:00:00
author: Fanteathy
tags: java
categories: java
---



基于pylon的服务,包括代码结构及开发流程。

sample可以参考[sample-project](docs/sample-project.zip), repo地址[https://git.elenet.me/pts/sample-project](https://git.elenet.me/pts/sample-project)

也可以参考[settle项目](https://git.elenet.me/fin/settle)

***最好的***参考项目[compensation-service](https://git.elenet.me/pts/compensation-service)

## 代码结构

- api: 接口定义相关
	- api: 接口定义
		- OrderApiService: 提供的eleme_order相关的api 
	- dto: 定义请求的数据结构和返回的数据结构(对应javabean)
		- ElemeOrder: `OrderApiService`中接口需要的请求和返回数据结构
	- form: 也可以使用dto定义返回的数据结构，form定义请求的数据结构 ***注意(important): 最好不要定义form的constructor***
		- enum: 对于其中请求参数常量的使用。***可以将常量定义在enum中***，传递的时候使用enum实例即可，使用enum表达的信息更为丰富 
	- exception: 接口exception定义
		- ServiceException: 表示业务相关的异常(如用户不存在，红包已过期等)
		- SystemException: 表示服务内部的异常(如数据库连接超时，redis服务不可用等)
		- java.lang.RuntimeException: 非受检异常, 计入熔断统计
		- 其余异常参考: [Pylon内置Rpc异常](http://wiki.ele.to:8090/pages/viewpage.action?pageId=20328819)
- dao: model定义,和DB table crud一一对应，供`service`中接口实现使用
	- ElemeOrderDao: eleme_order表对应的数据模型，和eleme_order表字段对应
	- resources: DB连接，DB事务等
- deploy:
	- 发布相关定义(发布相关变量等), 最终发布文件存储位置
- docs: 文档
- service: 接口实现,业务逻辑定义。可以加一层BIZ层, service抽象成webapi, biz写详细业务逻辑。依赖`api`和`dao`
	- conf: 主要的配置文件
		- Configure.json: 最重要的配置文件。
			- serverConf: 服务端配置
				- initializer: IServiceInitializer实现类类名
				- interfaces: 提供服务的接口名列表
			- clientConfs: 客户端配置, 有依赖服务则需要声明
				- interfaces: 调用的接口名列表
	- OrderApiServiceImpl: `OrderApiService`中各种接口的实现
	- soa: 实现Pylon接口
		- ServiceInitializer: 服务初始化接口, 服务启动的时候调用对应init方法
			- getImpl: 返回指定接口对应的实现实例(所以能够通过提供的interface名找到对应的方法实现类)
	- MainApplication: `程序启动入口`, 启动基于Pylon的服务(其实是调用`ServiceInitializer.init()`) 。也可以通过me.ele.core.container.Container作为MainClass启动。(start.sh中配置启动入口)
- common: 定义公共组件
- pom.xml: 父级pom定义  

## 开发流程:

- 定义接口: api定义, interface定义，写对应的dto
- DAO层: db table crud实现
- service: 业务逻辑
- test: optional
- 发布: 
	- eless配置替换代码中配置
	- ci
	- start.sh
	- accept request

## 测试

测试方法有以下几种:

- 在单元测试代码中测试
	- 测试其他服务接口，需要构造`client`从huskar获取服务列表，发起RPC请求
		- 参考`me.ele.pts.sample.test.base.TestBase`的定义
			- `ClientUtil.getContext().initClients("deploy/conf/configure.json");`初始化Configure.json中定义的客户端，所以可以在`SampleServiceTest`中直接调用`TimeLineSearchService timeLineSearchService =
                (TimeLineSearchService) ClientUtil.getContext().getClients(TimeLineSearchService.class);`构造`TimelineSearchService`的客户端（从alpha环境中获取配置），并调用对应接口
	- 测试本服务接口，直接测试Service层对应接口实现代码即可。不需要通过接口调用，方便debug
		- 参考`me.ele.pts.sample.test.base.TestBase`的定义
			- `new AnnotationConfigApplicationContext(MainApplication.class);`注入`MainApplication`中定义的Bean，所以可以在`SampleServiceTest`中调用`SampleApi sampleService = ApplicationContextUtil.context.getBean(SampleApi.class);`获取对应的Bean,直接调用对应的方法即可 
- RPC测试，不能很好地debug，因为是RPC请求
	- `自己构造client测试`，需要另外写代码构造client
	- `通过postman测试`, 本地需要启动服务，或者通过`json-rpc`协议使用postman测试alpha环境

### 从IDE启动服务

按照`fin.settle_job_build.yml`定义的顺序部署好文件后。

配置[Run/Debug Configurations]:

设置[Application] `fin.settle_job`的:

- `Main class`: `me.ele.fin.job.MainApplication`
- `Working directory`: `/Users/joshua/finance/settle/settle-job/settle-job-service/deploy`。定位到`deploy`目录下，因为经过`ci_success.sh`后，所有的`conf`和`lib`都位于此目录下
- `Use classpath of module`: `settle-job-service`

直接启动，启动过程中观察对应的日志

***启动后可以使用`postman`发送请求,在IDE中设置断点，直接进行Debug***

### POSTMAN使用方法如下

#### method

post: `http://vpca-fin-settle-query-1.vm.elenet.me:8092/rpc`

#### body

```
{
  "ver": "1.0",
  "soa": {"req":"12345","rpc":"clientAppId|1.4.5.2"},
  "context": {"type":"lpt"},
  "iface": "me.ele.fin.settlement.api.soa.ISettlementSampleApi",
  "method": "helloWorld",
  "args": {"form":"{\"name\":\"test\"}"},
  "metas": {}
}
```

#### response

```
{
  "ver": "1.0",
  "soa": {
    "req": "12345"
  },
  "result": "{\"message\":\"hello world test\"}",
  "ex": null
}
```

也可以在本地启动服务，进行测试，参考`App_id_build.yml`中的步骤:

```
cd settle
mvn clean package -U -f settle-settlement/pom.xml -DskipTests=true
chmod +x settle-settlement/settle-settlement-service/deploy/ci_success.sh
settle-settlement/settle-settlement-service/deploy/ci_success.sh
cd settle/settle-settlement/settle-settlement-service/deploy
sh start.sh
```

post: `http://localhost:8092/rpc`

body

```
{
  "ver": "1.0",
  "soa": {"req":"12345","rpc":"clientAppId|1.4.5.2"},
  "context": {"type":"lpt"},
  "iface": "me.ele.fin.settlement.api.soa.ISettlementService",
  "method": "processBizSettle",
  "args": {"form":"{\"restaurantId\":123, \"OrderId\": 123, \"transNo\": \"2016\", \"businessType\": 1, \"operateType\":1, \"comment\": \"hello\", \"amount\": 1.23, \"compensationRate\": 1.23}"},
  "metas": {}
}
```

response:

```
{
  "ver": "1.0",
  "soa": {
    "req": "12345"
  },
  "result": "null",
  "ex": null
}
```


## SOA调用

因为pylon初始化时已经`initClients`了，所以直接构造client即可: 

```
TimeLineSearchService timeLineSearchService =
                (TimeLineSearchService) ClientUtil.getContext().getClients(TimeLineSearchService.class);
```

直接调用对应接口即可。

## 发布相关

发布最主要的脚本为位于根目录下的`$appid_build.yml`文件

具体过程参考[$appid_build.yml文件说明](docs/kuangjiagongju-39677747-071216-1111-18.pdf)

***可以在本地运行文件中相关命令以验证***

以`fin.settle_clearing_build.yml`为例:

- `mvn clean package -U -f pom.xml -DskipTests=true` 构建所有jar包，并跳过测试(使用`settle/pom.xml`) 

	`build`节点意义如下:

		1. `maven-compiler-plugin`: 生成java包
		2. `maven-source-plugin`: 生成sources源码包
	
此时会生成`module`下定义的所有包

```
<modules>
    <module>bill-query</module>
    <module>settle-clearing</module>
    <module>settle-dataretry</module>
    <module>settle-datasync</module>
    <module>settle-job</module>
    <module>settle-query</module>
    <module>settle-settlement</module>
</modules>
```

生成日志如下:

```
[INFO] settle ............................................. SUCCESS [  0.685 s]
[INFO] bill-query ......................................... SUCCESS [  2.565 s]
[INFO] settle-clearing .................................... SUCCESS [  0.009 s]
[INFO] settle-clearing-api ................................ SUCCESS [  0.342 s]
[INFO] settle-clearing-service ............................ SUCCESS [  2.688 s]
[INFO] settle-dataretry ................................... SUCCESS [  0.151 s]
[INFO] settle-datasync .................................... SUCCESS [  0.144 s]
[INFO] settle-job ......................................... SUCCESS [  0.141 s]
[INFO] settle-query ....................................... SUCCESS [  0.173 s]
[INFO] settle-settlement .................................. SUCCESS [  0.158 s]
```

对应的目录如下:

```
.
├── README.md
├── bill-query
│   ├── bill-query.iml
│   ├── pom.xml
│   ├── src
│   └── target
├── fin.settle_clearing_build.yml
├── pom.xml
├── settle-clearing
│   ├── pom.xml
│   ├── settle-clearing-api
│   ├── settle-clearing-service
│   └── settle-clearing.iml
├── settle-dataretry
│   ├── pom.xml
│   ├── settle-dataretry.iml
│   ├── src
│   └── target
├── settle-datasync
│   ├── pom.xml
│   ├── settle-datasync.iml
│   ├── src
│   └── target
├── settle-job
│   ├── pom.xml
│   ├── settle-job.iml
│   ├── src
│   └── target
├── settle-query
│   ├── pom.xml
│   ├── settle-query.iml
│   ├── src
│   └── target
├── settle-settlement
│   ├── pom.xml
│   ├── settle-settlement.iml
│   ├── src
│   └── target
└── settle.iml
```

注意`settle-clearing`划分为了`settle-clearing-api`,`settle-clearing-service`两个Module

对应的jar包如下: 

```
.//bill-query/target/bill-query-1.0-SNAPSHOT-sources.jar
.//bill-query/target/bill-query-1.0-SNAPSHOT.jar
.//settle-clearing/settle-clearing-api/target/settle-clearing-api-1.0-SNAPSHOT-sources.jar
.//settle-clearing/settle-clearing-api/target/settle-clearing-api-1.0-SNAPSHOT.jar
.//settle-clearing/settle-clearing-service/target/settle-clearing-service-1.0-SNAPSHOT-sources.jar
.//settle-clearing/settle-clearing-service/target/settle-clearing-service-1.0-SNAPSHOT.jar
.//settle-dataretry/target/settle-dataretry-1.0-SNAPSHOT-sources.jar
.//settle-dataretry/target/settle-dataretry-1.0-SNAPSHOT.jar
.//settle-datasync/target/settle-datasync-1.0-SNAPSHOT-sources.jar
.//settle-datasync/target/settle-datasync-1.0-SNAPSHOT.jar
.//settle-job/target/settle-job-1.0-SNAPSHOT-sources.jar
.//settle-job/target/settle-job-1.0-SNAPSHOT.jar
.//settle-query/target/settle-query-1.0-SNAPSHOT-sources.jar
.//settle-query/target/settle-query-1.0-SNAPSHOT.jar
.//settle-settlement/target/settle-settlement-1.0-SNAPSHOT-sources.jar
.//settle-settlement/target/settle-settlement-1.0-SNAPSHOT.jar
```

- after_success

```
after_success:
   - chmod +x settle-clearing/settle-clearing-service/deploy/ci_success.sh
   - settle-clearing/settle-clearing-service/deploy/ci_success.sh
```

运行`ci_success.sh`，将`settle-clearing/settle-clearing-service`下面的zip文件(`settle-clearing-service.zip`)放在`deploy`目录下

`settle-clearing-service.zip`的生成依赖于`settle-clearing/settle-clearing-service/pom.xml`的定义， 其中的`build`节点定义了`编译打包配置`

`settle-clearing-service.zip`结构为

```
lib/
	...
	pylon-rpc-2.0.16.jar
	pylon-spring-2.0.16.jar
	settle-clearing-api-1.0-SNAPSHOT.jar
	settle-clearing-service-1.0-SNAPSHOT.jar
	sigar-1.6.4.jar
	slf4j-api-1.7.6.jar
	...
``` 

此时所有的待发布文件存储在deploy目录下，对应的结构为

```
deploy/
	lib/
		*.jar
	conf/
		*.json
		*.properties
```  
所有的配置文件也可以放在deploy目录下，使用基于classpath的目录获取文件路径`"classpath:/application.properties"`

也可以使用基于根目录(deploy)的方式获取路径(/conf),因为此时根目录即为deploy

- 最终部署

```
outfile:
   - settle-clearing/settle-clearing-service/deploy
```

发布系统将`deploy`下的所有目录抽取部署至`/data/appid`,并运行`start.sh`启动服务

其中`CLASS_PATH="$PROJECT_DIR/conf:$PROJECT_DIR/lib/*:$CLASS_PATH"`将`conf`,`lib`目录放入classpath中



## Pylon相关

### 原理

pylon原理: 提供对应的接口(interface)定义给pylon，服务方实现对应的接口。有对应的请求时，pylon会生成代理对象，调用service对这个接口的具体实现。


### 接口定义
 
 参照`transaction_scoring_system`
 
 由`Configure.json`中的`serverConf`提供对应SOA服务的接口定义
 
 
```
 "serverConf": {
    "name": "pts.score",
    "protocol": "json",
    "group": "local",
    "port": 8088,
    "threadPoolSize": 24,
    "bufferQueueSize": 30,
    "initializer": "me.ele.pts.score.impl.soa.TransactionScoringServiceInitializer",
    "interfaces": [
      "me.ele.pts.score.service.IRestaurantTransactionScore",
      "me.ele.pts.score.service.IUserTransactionScore"
    ]
  },
```
  
其中的`interfaces`定义好了对外提供的接口，同时注册在`huskar`的service中

此处接口定义在`me.ele.pts.api.OrderApiService`中

## Bean注入

`ServiceInitializer.init()`初始化资源。

`appContext = new AnnotationConfigApplicationContext(MainApplication.class);`会将`MainApplication.class`作为配置文件注入所有bean（因为`MainApplication`使用了`@Configuration`注解）

同时因为在`MainApplication`中定义了bean组件扫描的package: `@ComponentScan(basePackages = "me.ele.fin.settlement.impl")`,所以定义在`me.ele.fin.settlement.impl`下所有的bean会在初始化时注入IOC容器。


## 脚本

- crontab sh对应的java程序main函数（不推荐）
- ScheduledThreadPoolExecutor
- jobplus

## 参考

- [Pylon 2.0接入文档](http://wiki.ele.to:8090/pages/viewpage.action?pageId=20333808)或者参考[pdf版本](docs/kuangjiagongju-20333808-141116-1409-4.pdf)

## 其他

- 打包: 使用maven-assembly-plugin插件，配合dist.xml，conf目录存放配置文件，bin目录存放可执行脚本，lib目录存放所有依赖jar包。
- profile: 环境参数相关（和开发无关，略）
- 多module/多package的结构都可以