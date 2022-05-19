---
title: Seata-AT
date: 2022-05-19 19:49:31
tags:
- Seata
categories:
- SpringCloud
---


## AT模式两阶段提交

- 基于支持本地 ACID 事务的关系型数据库。
- Java 应用，通过 JDBC 访问数据库。


两阶段提交协议的演变：

- 一阶段：业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。
- 二阶段：
  - 提交异步化，非常快速地完成。
  - 回滚通过一阶段的回滚日志进行反向补偿。

隔离级别：全局事务隔离级别为读未提交

## 如何解决脏读脏写:

1：脏写因为出现在分支事务中，可使用`@GlobalTransactional` 升级分支事务请求全局事务锁，来解决脏写问题。

2：脏写因为出现在分支事务中，可使用`@GlobalLock + select for update` 升级分支事务请求全局事务锁，来解决脏写问题。

3：脏读因为全局事务有可能回滚，导致事务外查询数据会读取到脏数据，可使用`@GlobalLock + select for update` 升级分支事务请求全局事务锁，来解决脏读问题。




**允许空回滚：**

事务协调器在调用TCC服务的一阶段Try操作时，可能会出现因为丢包而导致的网络超时，此时事务协调器会触发二阶段回滚，调用TCC服务的Cancel操作；

TCC服务在未收到Try请求的情况下收到Cancel请求，这种场景被称为空回滚；TCC服务在实现时应当允许空回滚的执行；


**防悬挂控制:**

事务协调器在调用TCC服务的一阶段Try操作时，可能会出现因网络拥堵而导致的超时，此时事务协调器会触发二阶段回滚，调用TCC服务的Cancel操作；在此之后，拥堵在网络上的一阶段Try数据包被TCC服务收到，出现了二阶段Cancel请求比一阶段Try请求先执行的情况；

用户在实现TCC服务时，应当允许空回滚，但是要拒绝执行空回滚之后到来的一阶段Try请求。

**幂等控制**:
		无论是网络数据包重传，还是异常事务的补偿执行，都会导致TCC服务的Try、Confirm或者Cancel操作被重复执行；用户在实现TCC服务时，需要考虑幂等控制，即Try、Confirm、Cancel 执行次和执行多次的业务结果是一样的

**业务数据可见性控制:**

TCC服务的一阶段Try操作会做资源的预留，在二阶段操作执行之前，如果其他事务需要读取被预留的资源数据，那么处于中间状态的业务数据该如何向用户展示，需要业务在实现时考虑清楚；通常的设计原则是“宁可不展示、少展示，也不多展示、错展示”



**业务数据并发访问控制:**
		TCC服务的一阶段Try操作预留资源之后，在二阶段操作执行之前，预留的资源都不会被释放；如果此时其他分布式事务修改这些业务资源，会出现分布式事务的并发问题；

用户在实现TCC服务时，需要考虑业务数据的并发控制，尽量将逻辑锁粒度降到最低，以最大限度的提高分布式事务的并发性

## 全局事务锁

```java
//io.seata.rm.datasource.exec.BaseTransactionalExecutor类
//全局锁key构建 及锁请求检查
protected void prepareUndoLog(TableRecords beforeImage, TableRecords afterImage) throws SQLException {
        if (beforeImage.getRows().isEmpty() && afterImage.getRows().isEmpty()) {
            return;
        }
        ConnectionProxy connectionProxy = statementProxy.getConnectionProxy();
        TableRecords lockKeyRecords = sqlRecognizer.getSQLType() == SQLType.DELETE ? beforeImage : afterImage;
    	//构建全局事务锁key
        String lockKeys = buildLockKey(lockKeyRecords);
        connectionProxy.appendLockKey(lockKeys);
        SQLUndoLog sqlUndoLog = buildUndoItem(beforeImage, afterImage);
        connectionProxy.appendUndoLog(sqlUndoLog);
    }

//io.seata.rm.datasource.exec.SelectForUpdateExecutor
//全局锁key构建 及锁请求检查
public T doExecute(Object... args) throws Throwable {
                    // Try to get global lock of those rows selected
    TableRecords selectPKRows = buildTableRecords(getTableMeta(), selectPKSQL, paramAppenderList);
    //构建全局事务锁key
    String lockKeys = buildLockKey(selectPKRows);
    if (RootContext.inGlobalTransaction()) {
        //do as usual
        statementProxy.getConnectionProxy().checkLock(lockKeys);
    } else if (RootContext.requireGlobalLock()) {
        //check lock key before commit just like DML to avoid reentrant lock problem(no xid thus ca
        // not reentrant)
        statementProxy.getConnectionProxy().appendLockKey(lockKeys);
    } else {
        throw new RuntimeException("Unknown situation!");
    }
                  
}
//io.seata.rm.datasource.ConnectionProxy类
//提交前注册分支并申请及检查全局锁
private void doCommit() throws SQLException {
    if (context.inGlobalTransaction()) {
        //@GlobalTransaction
        processGlobalTransactionCommit();
    } else if (context.isGlobalLockRequire()) {
        //@GlobalLocks
        processLocalCommitWithGlobalLocks();
    } else {
        targetConnection.commit();
    }
}
    //@GlobalTransaction
    private void processGlobalTransactionCommit() throws SQLException {
                //注册分支并获取全局锁
                register();
          ...
        }
     //@GlobalLocks
    private void processLocalCommitWithGlobalLocks() throws SQLException {
            checkLock(context.buildLockKeys());
          ...
        }
    //事务分支注册并获取全局锁
     private void register() throws TransactionException {
         ...	
            Long branchId = 				       DefaultResourceManager.get().branchRegister(BranchType.AT,getDataSourceProxy().getResourceId(),
                null, context.getXid(), null, context.buildLockKeys());
		...
        }
	//全局锁检查
	public void checkLock(String lockKeys) throws SQLException {
        // Just check lock without requiring lock by now.
            boolean lockable = DefaultResourceManager.get().lockQuery(BranchType.AT,
                getDataSourceProxy().getResourceId(), context.getXid(), lockKeys);
            if (!lockable) {
                throw new LockConflictException();
            }
    }
```

[深度剖析 Seata TCC 模式（一）](https://seata.io/zh-cn/blog/seata-tcc.html)


