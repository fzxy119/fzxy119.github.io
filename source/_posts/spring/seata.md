---
title: Seata安装
date: 2022-05-19 19:47:29
tags:
- Seata
categories:
- SpringCloud
---
Seata 是一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata 将为用户提供了 AT、TCC、SAGA 和 XA 事务模式，为用户打造一站式的分布式解决方案,



## 一:Seata术语

#### TC (Transaction Coordinator) - 事务协调者

维护全局和分支事务的状态，驱动全局事务提交或回滚。

#### TM (Transaction Manager) - 事务管理器

定义全局事务的范围：开始全局事务、提交或回滚全局事务。

#### RM (Resource Manager) - 资源管理器

管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。



## 二：资源介绍

Seata分TC、TM和RM三个角色，TC（Server端）为单独服务端部署，TM和RM（Client端）由业务系统集成。

#### client

存放client端sql脚本 (包含 **undo_log**表) 

#### config-center

各个配置中心参数导入脚本，**config.txt**(包含server和client，原名nacos-config.txt)为通用参数文件

#### server

server端数据库脚本 (**包含 lock_table、branch_table  global_table 与 distributed_lock**) 及各个容器配置

## 三：Server安装：

Server端存储模式（store.mode）现有file、db、redis三种（后续将引入raft,mongodb），

file模式无需改动，直接启动即可，注： file模式为单机模式，全局事务会话信息内存中读写并持久化本地文件root.data，性能较高;

db模式为高可用模式，全局事务会话信息通过db共享，相应性能差些;

redis模式Seata-Server 1.3及以上版本支持,性能较高,存在事务信息丢失风险,请提前配置合适当前场景的redis持久化配置.

### 文件配置方式

#### 文件模式

##### file.config 配置存储模式  

此处配置了全局事务以何种方式来存储，配置mode = "db" 支持 file db redis

```json
store {
  ## store mode: file、db、redis
  mode = "db"
  ## rsa decryption public key
  publicKey = ""
  ## file store property
  file {
    ## store location dir
    dir = "sessionStore"
    # branch session size , if exceeded first try compress lockkey, still exceeded throws exceptions
    maxBranchSessionSize = 16384
    # globe session size , if exceeded throws exceptions
    maxGlobalSessionSize = 512
    # file buffer size , if exceeded allocate new buffer
    fileWriteBufferCacheSize = 16384
    # when recover batch read size
    sessionReloadReadSize = 100
    # async, sync
    flushDiskMode = async
  }

  ## database store property
  db {
    ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
    datasource = "druid"
    ## mysql/oracle/postgresql/h2/oceanbase etc.
    dbType = "mysql"
    driverClassName = "com.mysql.jdbc.Driver"
    ## if using mysql to store the data, recommend add rewriteBatchedStatements=true in jdbc connection param
    url = "jdbc:mysql://127.0.0.1:3306/seata?rewriteBatchedStatements=true"
    user = "root"
    password = "CHENxiao1989119"
    minConn = 5
    maxConn = 100
    globalTable = "global_table"
    branchTable = "branch_table"
    lockTable = "lock_table"
    queryLimit = 100
    maxWait = 5000
  }

  ## redis store property
  redis {
    ## redis mode: single、sentinel
    mode = "single"
    ## single mode property
    single {
      host = "127.0.0.1"
      port = "6379"
    }
    ## sentinel mode property
    sentinel {
      masterName = ""
      ## such as "10.28.235.65:26379,10.28.235.65:26380,10.28.235.65:26381"
      sentinelHosts = ""
    }
    password = ""
    database = "0"
    minConn = 1
    maxConn = 10
    maxTotal = 100
    queryLimit = 100
  }
}
```

##### registry.conf 注册及配置中心

此处配置seata的配置中心使用何种方式，及集群模式下的seata服务如何注册服务

####  使用nacos作为配置中心及服务治理中心

注册中心type = "nacos"，seata服务将seataServer注册到注册中心

配置中心type = "nacos"   seata服务将从nacos取得seata服务的配置信息，当采用file模式，将从file.conf文件中获取配置

seata服务配置项可从config.txt,中获取，并上传至配置中心

（1）下载config.txt,并自行对应的脚本nacos-config.sh 会到上层目录查找config.txt

```shell
sh /d/softw/seata/seata-server-1.4.2/conf/nacos-config.sh -h localhost -p 8848 -g SEATA_GROUP -t 7dba3cc6-3805-4f3d-b89c-0519044013b7 -u username -w password
-h -- nacos ip
-p -- nacos 端口
-g -- seata-server配置文件 分组名
-t -- seata-server配置文件 命名空间ID
-u -- nacos账号
-w -- nacos密码

```

（2）server可使用dataId = "seataServer.properties" cleant端使用上传配置 并拷贝参数到nacos

```json
nacos {
    serverAddr = "127.0.0.1:8848"
    namespace = "7dba3cc6-3805-4f3d-b89c-0519044013b7"
    group = "SEATA_GROUP"
    username = ""
    password = ""
    #可指定dataId
    dataId = "seataServer.properties"
  }
```



### DBScript脚本

#### Server端 MYSQL

```sql
-- -------------------------------- The script used when storeMode is 'db' --------------------------------
-- the table to store GlobalSession data
CREATE TABLE IF NOT EXISTS `global_table`
(
    `xid`                       VARCHAR(128) NOT NULL,
    `transaction_id`            BIGINT,
    `status`                    TINYINT      NOT NULL,
    `application_id`            VARCHAR(32),
    `transaction_service_group` VARCHAR(32),
    `transaction_name`          VARCHAR(128),
    `timeout`                   INT,
    `begin_time`                BIGINT,
    `application_data`          VARCHAR(2000),
    `gmt_create`                DATETIME,
    `gmt_modified`              DATETIME,
    PRIMARY KEY (`xid`),
    KEY `idx_gmt_modified_status` (`gmt_modified`, `status`),
    KEY `idx_transaction_id` (`transaction_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store BranchSession data
CREATE TABLE IF NOT EXISTS `branch_table`
(
    `branch_id`         BIGINT       NOT NULL,
    `xid`               VARCHAR(128) NOT NULL,
    `transaction_id`    BIGINT,
    `resource_group_id` VARCHAR(32),
    `resource_id`       VARCHAR(256),
    `branch_type`       VARCHAR(8),
    `status`            TINYINT,
    `client_id`         VARCHAR(64),
    `application_data`  VARCHAR(2000),
    `gmt_create`        DATETIME(6),
    `gmt_modified`      DATETIME(6),
    PRIMARY KEY (`branch_id`),
    KEY `idx_xid` (`xid`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

-- the table to store lock data
CREATE TABLE IF NOT EXISTS `lock_table`
(
    `row_key`        VARCHAR(128) NOT NULL,
    `xid`            VARCHAR(128),
    `transaction_id` BIGINT,
    `branch_id`      BIGINT       NOT NULL,
    `resource_id`    VARCHAR(256),
    `table_name`     VARCHAR(32),
    `pk`             VARCHAR(36),
    `status`         TINYINT      NOT NULL DEFAULT '0' COMMENT '0:locked ,1:rollbacking',
    `gmt_create`     DATETIME,
    `gmt_modified`   DATETIME,
    PRIMARY KEY (`row_key`),
    KEY `idx_status` (`status`),
    KEY `idx_branch_id` (`branch_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

CREATE TABLE IF NOT EXISTS `distributed_lock`
(
    `lock_key`       CHAR(20) NOT NULL,
    `lock_value`     VARCHAR(20) NOT NULL,
    `expire`         BIGINT,
    primary key (`lock_key`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4;

INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('AsyncCommitting', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryCommitting', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('RetryRollbacking', ' ', 0);
INSERT INTO `distributed_lock` (lock_key, lock_value, expire) VALUES ('TxTimeoutCheck', ' ', 0);
```

#### Client端sql

##### AT模式 MYSQL AT模式支持的数据库有：MySQL、Oracle、PostgreSQL、 TiDB、MariaDB。

```sql
-- 注意此处0.7.0+ 增加字段 context
-- for AT mode you must to init this sql for you business database. the seata server not need it.
CREATE TABLE IF NOT EXISTS `undo_log`
(
    `branch_id`     BIGINT       NOT NULL COMMENT 'branch transaction id',
    `xid`           VARCHAR(128) NOT NULL COMMENT 'global transaction id',
    `context`       VARCHAR(128) NOT NULL COMMENT 'undo_log context,such as serialization',
    `rollback_info` LONGBLOB     NOT NULL COMMENT 'rollback info',
    `log_status`    INT(11)      NOT NULL COMMENT '0:normal status,1:defense status',
    `log_created`   DATETIME(6)  NOT NULL COMMENT 'create datetime',
    `log_modified`  DATETIME(6)  NOT NULL COMMENT 'modify datetime',
    UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8 COMMENT ='AT transaction mode undo table';
```

##### SAGE模式 MYSQL Saga模式不依赖数据源。

```sql
-- -------------------------------- The script used for sage  --------------------------------


CREATE TABLE IF NOT EXISTS `seata_state_machine_def`
(
    `id`               VARCHAR(32)  NOT NULL COMMENT 'id',
    `name`             VARCHAR(128) NOT NULL COMMENT 'name',
    `tenant_id`        VARCHAR(32)  NOT NULL COMMENT 'tenant id',
    `app_name`         VARCHAR(32)  NOT NULL COMMENT 'application name',
    `type`             VARCHAR(20)  COMMENT 'state language type',
    `comment_`         VARCHAR(255) COMMENT 'comment',
    `ver`              VARCHAR(16)  NOT NULL COMMENT 'version',
    `gmt_create`       DATETIME(3)  NOT NULL COMMENT 'create time',
    `status`           VARCHAR(2)   NOT NULL COMMENT 'status(AC:active|IN:inactive)',
    `content`          TEXT COMMENT 'content',
    `recover_strategy` VARCHAR(16) COMMENT 'transaction recover strategy(compensate|retry)',
    PRIMARY KEY (`id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

CREATE TABLE IF NOT EXISTS `seata_state_machine_inst`
(
    `id`                  VARCHAR(128)            NOT NULL COMMENT 'id',
    `machine_id`          VARCHAR(32)             NOT NULL COMMENT 'state machine definition id',
    `tenant_id`           VARCHAR(32)             NOT NULL COMMENT 'tenant id',
    `parent_id`           VARCHAR(128) COMMENT 'parent id',
    `gmt_started`         DATETIME(3)             NOT NULL COMMENT 'start time',
    `business_key`        VARCHAR(48) COMMENT 'business key',
    `start_params`        TEXT COMMENT 'start parameters',
    `gmt_end`             DATETIME(3) COMMENT 'end time',
    `excep`               BLOB COMMENT 'exception',
    `end_params`          TEXT COMMENT 'end parameters',
    `status`              VARCHAR(2) COMMENT 'status(SU succeed|FA failed|UN unknown|SK skipped|RU running)',
    `compensation_status` VARCHAR(2) COMMENT 'compensation status(SU succeed|FA failed|UN unknown|SK skipped|RU running)',
    `is_running`          TINYINT(1) COMMENT 'is running(0 no|1 yes)',
    `gmt_updated`         DATETIME(3) NOT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `unikey_buz_tenant` (`business_key`, `tenant_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;

CREATE TABLE IF NOT EXISTS `seata_state_inst`
(
    `id`                       VARCHAR(48)  NOT NULL COMMENT 'id',
    `machine_inst_id`          VARCHAR(128) NOT NULL COMMENT 'state machine instance id',
    `name`                     VARCHAR(128) NOT NULL COMMENT 'state name',
    `type`                     VARCHAR(20)  COMMENT 'state type',
    `service_name`             VARCHAR(128) COMMENT 'service name',
    `service_method`           VARCHAR(128) COMMENT 'method name',
    `service_type`             VARCHAR(16) COMMENT 'service type',
    `business_key`             VARCHAR(48) COMMENT 'business key',
    `state_id_compensated_for` VARCHAR(50) COMMENT 'state compensated for',
    `state_id_retried_for`     VARCHAR(50) COMMENT 'state retried for',
    `gmt_started`              DATETIME(3)  NOT NULL COMMENT 'start time',
    `is_for_update`            TINYINT(1) COMMENT 'is service for update',
    `input_params`             TEXT COMMENT 'input parameters',
    `output_params`            TEXT COMMENT 'output parameters',
    `status`                   VARCHAR(2)   NOT NULL COMMENT 'status(SU succeed|FA failed|UN unknown|SK skipped|RU running)',
    `excep`                    BLOB COMMENT 'exception',
    `gmt_updated`              DATETIME(3) COMMENT 'update time',
    `gmt_end`                  DATETIME(3) COMMENT 'end time',
    PRIMARY KEY (`id`, `machine_inst_id`)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8;
```

##### TCC模式mysql   TCC模式不依赖数据源(1.4.2版本及之前)，1.4.2版本之后增加了TCC防悬挂措施，需要数据源支持。

```sql
-- -------------------------------- The script use tcc fence  --------------------------------
CREATE TABLE IF NOT EXISTS `tcc_fence_log`
(
    `xid`           VARCHAR(128)  NOT NULL COMMENT 'global id',
    `branch_id`     BIGINT        NOT NULL COMMENT 'branch id',
    `action_name`   VARCHAR(64)   NOT NULL COMMENT 'action name',
    `status`        TINYINT       NOT NULL COMMENT 'status(tried:1;committed:2;rollbacked:3;suspended:4)',
    `gmt_create`    DATETIME(3)   NOT NULL COMMENT 'create time',
    `gmt_modified`  DATETIME(3)   NOT NULL COMMENT 'update time',
    PRIMARY KEY (`xid`, `branch_id`),
    KEY `idx_gmt_modified` (`gmt_modified`),
    KEY `idx_status` (`status`)
) ENGINE = InnoDB
DEFAULT CHARSET = utf8;
```



##### XA模式   XA模式只支持实现了XA协议的数据库。Seata支持MySQL、Oracle、PostgreSQL和MariDB。

## 四：事务分组

- 每个Seata应用侧的RM、TM，都具有一个**事务分组**名

- 每个Seata协调器侧的TC，都具有一个**集群名**和**Seata服务地址列表** 映射

  ​    应用侧连接协调器侧时，经历如下两步：

  ​	通过事务分组的名称，从配置中获取到该应用侧对应的TC集群名

  ​	过集群名称，可以从注册中心中获取TC集群的地址列表 

```properties
#Client端配置事务分组
seata.tx-service-group=${spring.application.name}-tx-group
#Server端配置事务分组与集群的映射关系
service.vgroupMapping.default_tx_group=default
service.vgroupMapping.seata-order-tx-group=default
service.vgroupMapping.seata-store-tx-group=default
service.vgroupMapping.seata-shop-tx-group=default

```

Seata集群名称是在SeataServer registry.conf配置文件中指定

```json
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"
  nacos {
    application = "seata-server"
    serverAddr = "127.0.0.1:8848"
    group = "SEATA_GROUP"
    namespace = "7dba3cc6-3805-4f3d-b89c-0519044013b7"
    cluster = "default"
    username = ""
    password = ""
  }
}
```



## 远程服务调用XID的事务传播：

分布在网络上的服务如何感知并加入到分布式的事务过程中呢，事务属性如何传播的呢？


**在spring-cloud-starter-alibaba-seata环境下调用方：**

Seata集成了RestTemplet 并自动配了com.alibaba.cloud.seata.SeataRestTemplateAutoConfiguration

Seata集成了RestTemplet 并自动配了com.alibaba.cloud.seata.feign.SeataFeignClientAutoConfiguration

Seata集成了hystrix 并自动配了com.alibaba.cloud.seata.feign.hystrix.SeataHystrixAutoConfiguration

```java
io.seata.integration.http.TransactionPropagationIntercepter{
public ClientHttpResponse intercept(HttpRequest httpRequest, byte[] bytes, ClientHttpRequestExecution clientHttpRequestExecution) throws IOException {
        HttpRequestWrapper requestWrapper = new HttpRequestWrapper(httpRequest);
        String xid = RootContext.getXID();
        if (!StringUtils.isEmpty(xid)) {
            #将TX_XID 添加到请求头
            requestWrapper.getHeaders().add("TX_XID", xid);
        }
        return clientHttpRequestExecution.execute(requestWrapper, bytes);
    }
}

com.alibaba.cloud.seata.feign.SeataFeignClient{
    public Response execute(Request request, Options options) throws IOException {
        #修改请求头
        Request modifiedRequest = this.getModifyRequest(request);
        return this.delegate.execute(modifiedRequest, options);
    }

    private Request getModifyRequest(Request request) {
        String xid = RootContext.getXID();
        if (StringUtils.isEmpty(xid)) {
            return request;
        } else {
            Map<String, Collection<String>> headers = new HashMap(16);
            headers.putAll(request.headers());
            List<String> seataXid = new ArrayList();
            seataXid.add(xid);
            #添加TX_XID到请求头
            headers.put("TX_XID", seataXid);
            return Request.create(request.method(), request.url(), headers, request.body(), request.charset());
        }
    }   
}
```

**在spring-cloud-starter-alibaba-seata环境下接收方：**

Seata集成了MvcHandlerInterceptor并自动配了com.alibaba.cloud.seata.SeataHandlerInterceptorConfiguration

```java
public class SeataHandlerInterceptor implements HandlerInterceptor {
    private static final Logger log = LoggerFactory.getLogger(SeataHandlerInterceptor.class);

    public SeataHandlerInterceptor() {
    }

    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String xid = RootContext.getXID();
        String rpcXid = request.getHeader("TX_XID");
        if (log.isDebugEnabled()) {
            log.debug("xid in RootContext {} xid in RpcContext {}", xid, rpcXid);
        }

        if (StringUtils.isBlank(xid) && rpcXid != null) {
            #从请求头中拿到TX_XID 并绑定rpcxid 达到事务传播功能
            RootContext.bind(rpcXid);
            if (log.isDebugEnabled()) {
                log.debug("bind {} to RootContext", rpcXid);
            }
        }

        return true;
    }

}
```

**注：在seata-all包io.seata.integration中 集成了，dubbo,grpc,http,motan,sofa.rpc等事务传播功能：**



## 事务边界定义、控制及事务状态查询（高阶api）

### GlobalTransactionScanner(AbstractAutoProxyCreator),（注解扫描）

代理对象创建过程,对Seata目标代理对象进行分析并创建合适的植入点。确定哪些类可以做TCC,,根据目标类配置是否是TCC模式 ，确定是否植入TccActionInterceptor拦截（如类上是否有@LocalTCC），根据类上是否有@GlobalTransactional 或者方法上是否有	@GlobalTransactional 或者@GlobalLock 来确定是否植入GlobalTransactionalInterceptor。

TccActionInterceptor对指定@LocalTCC的类方法上有@TwoPhaseBusinessAction做拦截

```
@LocalTCC
public interface OrderTcc {
    @TwoPhaseBusinessAction(name = "insertOrder",commitMethod = "commit",rollbackMethod = "cancel")
    public Object insertOrder(@BusinessActionContextParameter(paramName = "order") OrderInfo order);
    public boolean commit(BusinessActionContext businessActionContext);
    public boolean cancel(BusinessActionContext businessActionContext);
}
```

GlobalTransactionalInterceptor(MethodInterceptor) 提供两种模板方法，根据注解的不同使用不同的模板方法

​	1：TransactionalTemplate 请求全局锁，开启全局事务 模板方法

​		通过GlobalTransactionContext GlobalTransaction  把一个业务服务的调用包装成带有分布式事务支持的服务。

​	2：GlobalLockTemplate，请求全局锁

​		通过RootContext 把一个业务服务的调用包装全局事务锁内进行的服务。

## 用于控制事务上下文的传播(低阶api)

RootContext 事务的根上下文：负责在应用的运行时，维护 XID 。

高阶API 的实现都是基于 RootContext 中维护的 XID 来做的。应用的当前运行的操作是否在一个全局事务的上下文中，就是看 RootContext 中是否有 XID，RootContext 的默认实现是基于 ThreadLocal 的，即 XID 保存在当前线程上下文中

Seata管方文档:[Seata](https://seata.io/zh-cn/index.html)


