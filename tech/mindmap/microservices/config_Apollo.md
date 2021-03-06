# Apollo

## 业务需求

### 例：实现限购功能

#### 1. 代码写死

### 痛点

#### 静态配置在运行时无法动态修改

#### 配置散乱，格式不标准

#### 易引发生产事故：容易将非生产配置带到生产上

#### 配置修改麻烦

#### 缺少安全审计

#### 缺少版本控制

### 核心需求

#### 交付件和配置分离

#### 抽象标准化：用户不用关心格式

#### 集中式

#### 高可用

#### 实时性

#### 治理

## 基础

### 配置

#### 定义

##### 可独立于程序的可配变量

##### 例如连接字符串、应用配置、业务配置

#### 形态

##### 程序hard code

##### 配置文件

##### 环境变量

##### 启动参数

##### 基于数据库

#### 治理

##### 权限控制、审计

##### 不同环境、集群配置管理

##### 框架类组件配置管理

##### 灰度发布

#### 分类

##### 静态配置

###### 环境相关

####### 数据库/中间件/其他服务的连接字符串

###### 安全配置

####### 用户名，密码，令牌，许可证书

##### 动态配置

###### 应用配置

####### 请求超时，线程池、队列、缓存、数据库连接池容量，日志级别，限流熔断阈值，黑白名单

###### 功能开关

####### 蓝绿发布，灰度开关，降级开关，HA高可用开关，DB迁移

DB迁移：

https://blog.launchdarkly.com/feature-flagging-to-mitigate-risk-in-database-migration/

####### 开关驱动开发（Feature Flag Driven Development）

https://blog.launchdarkly.com/feature-flag-driven-development/

####### TBD, Trunk Based Development

https://www.continuousdeliveryconsulting.com/blog/organisat

######## 新功能代码隐藏在功能开关后面

###### 业务配置

####### 促销规则，贷款额度、利率

### Apollo 功能亮点

#### 统一管理配置

#### 管理不同环境、不同集群的配置

#### 实时生效

#### 版本管理

#### 灰度发布

#### 权限：发布审核、操作审计

#### 客户端配置信息监控

## 设计

### 核心概念

#### application

##### 表示使用配置的应用

##### 代码相关：classpath:/META-INF/app.properties -> appid

#### environment

##### 表示配置对应的环境：DEV, FAT, UAT, PRO

##### 代码无关：/opt/settings/server.properties -> env

#### cluster

##### 表示一个应用下不同实例的分组

##### 代码无关：/opt/settings/server.properties -> idc

#### namespace

##### 表示一个应用下不同配置的分组：数据库配置、服务框架配置、

##### 应用默认有自己的名字空间：application

##### 类型

###### private

####### 只能被所属应用获取

###### public

####### 共享配置

###### 关联类型（继承类型）

####### 对公共组件的配置进行调整

##### 类别

###### properties

每一行配置对应一条 Item 记录

###### yml, yaml

一个 Namespace 仅仅对应一条 Item 记录。

###### json

###### xml

#### item

##### 表示可配置项

##### 格式

###### properties

###### json

###### xml

##### 定位方式

###### private: env + app + cluster + namespace + itemKey

###### public: env + cluster + namespace + itemKey

### 组件

#### Config Service

##### 配置读取、推动

##### 服务Apollo客户端

##### 长连接接口：Spring DeferedResult

#### Admin Service

##### 配置修改、发布

##### 服务Apollo Portal

#### Portal

##### 配置管理界面

##### 客户端软负载

#### Client

##### 长连接实时获取更新：Http Long Polling

##### 定时拉取配置：fallback

##### 本地缓存

##### 客户端软负载

#### Eureka

##### Config/Admin Service注册并报心跳

##### 和Config Service一起部署

#### Meta Server

##### client通过meta server获取config service服务列表

##### portal通过meta server获取admin service服务列表

##### 相当于Eureka Proxy，封装Eureka服务发现接口

##### 和config service一起部署

### 实时推送设计

#### Portal -> Admin service 

#### Admin service -> Config service

##### 定时扫描ReleaseMessage表

#### Config service -> Client

##### 长连接 Spring DeferedResult

### 客户端设计

#### 推拉结合

##### 保持一个长连接，配置实时推送

##### 定期拉配置（fallback）

#### 缓存

##### 内存缓存

##### 本地文件缓存 

#### 应用程序

##### 通过Apollo Client获取最新配置

##### 订阅配置更新通知

## 使用

### 客户端

#### 获取

Config config = ConfigService.getAppConfig();
Integer defaultRequestTimeout = 200;
Integer requestTimeout = config.getIntProperty("requestTimeout", defaultRequestTimeout);

#### 监听

```java
Config config = ConfigService.getAppConfig();

config.addChangeListener(new ConfigChangeListener() {

  @Override
  public void onChange(ConfigChangeEvent changeEvent) {
    for (String key : changeEvent.changedKeys()) {
      ConfigChange change = changeEvent.getChange(key);
      System.out.println(String.format(
        "Found change - key: %s, oldValue: %s, newValue: %s, changeType: %s",
        change.getPropertyName(), change.getOldValue(),
        change.getNewValue(), change.getChangeType()));
     }
  }
});
```

#### 接入

##### appid配置

###### 推荐：/META-INF/app.properties -> appid

###### 启动参数 -Dapp.id=

##### metaService地址配置

###### 启动参数 -Ddev_meta=

###### 推荐：apollo-core.jar --> apollo-env.properties

###### classpath: app-env.properties

##### env配置

###### 启动参数 -Denv=dev

###### 环境变量 ENV

###### 推荐：配置文件 opt/settings/server.properties -> env=

###### 注意：env=Local用来本地研发

##### cluster配置（可选）

###### 启动参数 -Dapollo.cluster=

###### 推荐：配置文件 opt/settings/server.properties -> idc=

#### 本地缓存

##### 缓存路径

###### /opt/data/{appid}/config-cache

###### {appid}-{clulster}-{namespace}.properties

### 灰度发布

https://github.com/ctripcorp/apollo/wiki/Apollo%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97#%E4%BA%94%E7%81%B0%E5%BA%A6%E5%8F%91%E5%B8%83%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97

## 部署

### 手工部署

#### 创建ApolloPortalDB

#### 创建ApolloConfigDB

#### demo.sh: 修改db url

#### demo.sh 执行

#### portal: localhost:8070

### docker部署

#### docker-quick-start -> docker-compose up

#### docker exec -i apollo-quick-start /apollo-quick-start/demo.sh client

### 分布式部署

#### 视频教程

https://pan.baidu.com/s/1rUAphfVq9fnEMqRrscDk-w?errno=0&errmsg=Auth%20Login%20Sucess&&bduss=&ssnerror=0&traceid=#list/path=%2F%E5%88%B6%E4%BD%9C%E8%A7%86%E9%A2%91%2Fapollo&parentPath=%2F%E5%88%B6%E4%BD%9C%E8%A7%86%E9%A2%91

#### Portal Server

##### 生产环境部署，管理FAT/UAT/PRO

##### ApolloPortalDB

###### apolloportaldb.sql

###### mvn -N -Pportaldb flyway:migrate

###### ServerConfig表

####### apollo.portal.envs

####### organizations

####### superAdmin

#### Meta/Conifg/Admin

##### 每个环境一套

##### Meta Server + Config Server 一个进程

##### Admin Server 一个进程

##### ApolloConfigDB

###### apolloconfigdb.sql

###### mvn -N -Pconfigdb flyway:migrate

###### ServerConfig表

####### eureka.service.url 

## 二次开发

### LDAP 登录

https://github.com/ctripcorp/apollo/wiki/Portal-%E5%AE%9E%E7%8E%B0%E7%94%A8%E6%88%B7%E7%99%BB%E5%BD%95%E5%8A%9F%E8%83%BD


## 源码

### 创建

#### 创建App

http://www.iocoder.cn/Apollo/portal-create-app/

##### Portal: AppController

###### AppService

####### 是否已存在？appRepository.findByAppId

####### 创建：AppRepository

####### 创建AppNS: AppNamespaceService

####### initAppRoles

####### Tracer.logEvent: CREATE_APP

###### EventPublisher

####### > CreationListener

######## AdminServiceAPI.crreatAPP

######### 同步到Config DB

########## Q: 如何保证一致性？

######### RetryableRestTemplate

###### RolePermission

##### Admin: AppController

###### 是否已存在？appService.findOne

###### 创建：adminService.createNewApp

####### 保存：appService.save

######## appRepository

######## auditService 审计入库

####### APP默认NS: appNamespaceService.createDefaultAppNamespace

####### APP默认集群：clusterService.createDefaultCluster

####### 集群默认NS：namespaceService.instanceOfAppNamespaces

#### 创建Cluster

http://www.iocoder.cn/Apollo/portal-create-cluster/

##### Portal: ClusterController

###### ClusterService

####### AdminServiceAPI.ClusterAPI#create

######## 同步到Config DB

####### 注意这里并没有保存到Portal DB!

####### Tracer.logEvent: CREATE_CLUSTER

##### Admin: ClusterController

###### ClusterService

####### 创建Cluster

####### 创建NS

#### 创建AppNamespace

http://www.iocoder.cn/Apollo/portal-create-namespace/

##### Portal: NamespaceController

###### @PreAurthorize

###### AppNamespaceService

####### appNamespaceRepository

####### 保存到PortalDB

###### assignNamespaceRoleToOperator

###### publishEvent: AppNamespaceCreationEvent

####### CreationListener#onAppNamespaceCreationEvent

####### AdminServiceAPI.NamespaceAPI

####### 同步到ConfigDB

##### Admin: AppNamespaceController

###### 校验重复

###### 存盘

#### 关联Namespace

http://www.iocoder.cn/Apollo/portal-associate-namespace/

##### Portal: NamespaceController

###### NamespaceService

####### AdminServiceAPI.NamespaceAPI

####### 保存到ConfigDB

##### Admin: NamespaceController

#### 创建Item

http://www.iocoder.cn/Apollo/portal-create-item/

##### Portal: ItemController

/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/item

###### ConfigService.createItem

####### AdminServiceAPI.ItemAPI

####### 保存到ConfigDB

####### Tracer.logEvent: MODIFY_NAMESPACE

##### Admin: ItemController

/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/items

###### ItemService

####### 校验重复

####### 保存

###### CommitService

### 权限

#### 限制修改人

##### @PreAcquireNamespaceLock

##### NamespaceAcquireLockAspect

##### NamespaceUnlockAspect

#### 限制发布人

##### ReleaseService.publish()

###### checkLock

### 发布配置

#### Portal: ReleaseController

/apps/{appId}/envs/{env}/clusters/{clusterName}/namespaces/{namespaceName}/releases	

##### ReleaseService.publish

###### AdminServiceAPI.ReleaseAPI

###### Tracer.logEvent: RELEASE_NAMESPACE

##### ApplicationEventPublisher: ConfigPublishEvent

#### Admin: ReleaseController

/apps/{appId}/clusters/{clusterName}/namespaces/{namespaceName}/releases

##### ReleaseService.publish

###### checkLock

###### getNamespaceItems --> 获取Item map

###### findLatestActiveRelease：获取上一次发布

###### 正常发布：masterRelease

####### createRelease

####### createReleaseHistory

###### 若为灰度：则mergeFromMasterAndPublishBranch

##### find cluster name

###### 子Namespace：说明是灰度发布

##### MessageSender

messageSender.sendMessage(

ReleaseMessageKeyGenerator.generate(appId, messageCluster, namespaceName),
                              Topics.APOLLO_RELEASE_TOPIC);

###### 基于数据库ReleaseMessage表实现的队列

Config Service 有一个线程会每秒扫描一次 ReleaseMessage 表，看看是否有新的消息记录

###### DatabaseMessageSender

####### Tracer.logEvent: ReleaseMessage

####### Tracer.newTransaction

Tracer.newTransaction("Apollo.AdminService", "sendMessage")


####### releaseMessageRepository.save()

####### BlockingQueue<Long> toClean.offer()

###### 持续清理ReleaseMessage

#######  DatabaseMessageSender.cleanExecutorService

####### 删除 releaseMessageRepository.findFirst100ByMessageAndIdLessThanOrderByIdAsc(

#### Config Service定期扫描

##### MessageScannerConfiguration

##### ReleaseMessageScanner

###### 获取bizConfig扫描频率

###### 获取最大ReleaseMessage编号

###### scheduleWithFixedDelay

####### scanAndSendMessages

######## 获取上次扫描后的最多500条记录

######## ReleaseMessageListener.handleMessage

######### 参见下一条

#### Config Service通知配置变化

##### NotificationControllerV2

###### implements ReleaseMessageListener

##### 处理客户端轮询

###### 接口

####### /notifications/v2

####### 返回值：DeferredResult

DeferredResult<ResponseEntity<List<ApolloConfigNotification>>> 

###### 流程

####### 过滤 ApolloConfigNotification Map

######## normalized归一化处理ns：大小写统一

会存储到DeferredResultWrapper中，以便返回结果时找到归一化之前的namespace

######## 去除.properties结尾

####### 组装 Watch Key Multimap

######## WatchKeysUtil

######## key: Namespace name, value: watch key

####### 获取最新 ReleaseMessage 列表

######## ReleaseMessageServiceWithCache

####### 获取最新 ApolloConfigNotification 列表

######## 遍历namespace集合

######## 判断是否有更新：服务端通知编号 是否大于客户端通知编号

####### 如有新ApolloConfigNotification，设置结果，结束长轮询

####### 如无新ApolloConfigNotification，注册deferredResults，等待配置更新通知

##### 监听配置变更 handleMessage()

###### 获取被修改的Namespace

###### 获取并创建DeferredResultWrapper 列表

####### 避免并发问题？

###### largeNotificationBatchExecutorService异步通知

###### DeferredResultWrapper.setResult 结束长轮询

### 灰度发布

### 客户端源码

#### Client轮询配置

##### 定时轮询：RemoteConfigRepository

###### Config Service: ConfigController

/configs/{appId}/{clusterName}/{namespace:.+}

####### 前置处理

######## 移除.properties

######## normalize

####### 获取Release：ConfigService.loadConfig

######## 灰度 > 普通

######## 指定cluster > IDC cluster > 默认 cluster

####### 获取公共类型NS下的Release

####### 后置处理

######## audit release

######## Tracer.logEvent: Apollo.Config.Found

###### 初始化

####### trySync

######## 初次配置初始化缓存

######## loadApolloConfig

######### 限流

######### getConfigServices

######### assembleQueryConfigUrl

######### doGet

####### schedulePeriodicRefresh

######## 定时刷新配置：定期sync()

####### scheduleLongPollingRefresh

######## 把自己注册到RemoteConfigLongPollService，实现实时通知

######## onLongPollNotified()

######### 同步配置

######### sync()

##### 长轮询：RemoteConfigLongPollService

###### 目的

####### 长轮询 /notifications/v2 接口

####### 如果新通知，则触发RemoteConfigRepository

###### doLongPollingRefresh

####### 限流

####### 获取config service 地址

####### doGet

####### 如果有新通知

######## updateNotifications

######## updateRemoteNotifications

######## notify()

######### RemoteConfigRepository. onLongPollNotified()

####### 如果没有新通知

######## 重置config service地址

######### 实现下次请求不同的config service

## 参考

### 官网

https://github.com/ctripcorp/apollo/wiki/Apollo%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97  

### 源码

http://www.iocoder.cn/Apollo  

#### 本地开发环境搭建

https://github.com/ctripcorp/apollo/wiki/Apollo%E5%BC%80%E5%8F%91%E6%8C%87%E5%8D%97

### 架构解析

https://mp.weixin.qq.com/s/-hUaQPzfsl9Lm3IqQW3VDQ

#### 简化架构图

### 使用场景

https://github.com/ctripcorp/apollo-use-cases
