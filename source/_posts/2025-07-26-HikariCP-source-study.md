---
title: HikariCP源码研读
tags:
  - java
  - 源码
  - 数据库连接池
category: 源码研读
keywords: HikariCP
description: HikariCP源码研读
cover: https://t13max.github.io/image/post/HikariCP.png
#cover: /image/post/HikariCP.png
date: 2025-07-26 14:08:37
---

# HikariCP源码研读

## 创建的流程

- 新建HikariConfig -> 新建HikariDataSource -> getConnection

## 重点类及说明

| 类名                       | 说明                                                             |
|--------------------------|----------------------------------------------------------------|
| `HikariDataSource`       | 对外数据源接口 负责对外暴露连接获取功能 管理连接池生命周期                                 |
| `HikariConfig`           | 加载配置、初始化参数                                                     |
| `HikariPool`             | 连接池核心，维护连接的创建、回收、驱逐和调度                                         |
| `PoolEntry`              | 连接池中的连接包装，保存连接状态和元数据和定时任务和Statement列表                          |
| `ConcurrentBag`          | 并发连接管理容器                                                       |
| `ProxyConnection`        | 对JDBC的连接的代理，增强功能（如泄漏检测、状态管理）                                   |
| `ProxyStatement`         | 对JDBC的语句的代理，增强功能（如泄漏检测、状态管理）                                   |
| `ProxyResultSet`         | 对JDBC的结果集的代理，增强功能（如泄漏检测、状态管理）                                  |
| `ConcurrentBag<T>`       | 存PoolEntry的地方 高性能容器 共享列表 ThreadLocal SynchronousQueue FastList |
| `FastList`               | 没有边界检测的高性能List                                                 |
| `SynchronousQueue`       | 没有缓冲区的 避免加锁和上下文切换 减少内存开销                                       |
| `ProxyLeakTask`          | 连接泄漏检测任务 超过一定时间没归还认为是泄漏                                        |
| `SuspendResumeLock`      | 连接池关闭或重配置期间阻止线程获取连接的工具类                                        |
| `ProxyFactory`           | (通过Javassist生成实现)生成各种代理实例，注入增强代码                               |
| `HouseKeeper`            | 周期性维护池状态 回收过期连接 补充新连接                                          |
| `JavassistProxyFactory`  | 比动态代理快 减少反射开销 功能增强 避免冗长代码                                      |
| `MetricsTrackerDelegate` | 监控                                                             |

## 常用类详解

### HikariConfig

HikariCP的配置, HikariDatasource继承了这个类, 创建数据源的时候需要传入一个HikariConfig对象, 然后会拷贝到HikariDatasource对象的父类字段上

#### 基础配置

| 配置项               | 说明       | 建议      |
|-------------------|----------|---------|
| `jdbcUrl`         | 数据库连接URL | 必填      |
| `username`        | 数据库用户名   | 必填      |
| `password`        | 数据库密码    | 必填      |
| `driverClassName` | JDBC驱动类名 | 可选 自动推导 |

#### 连接池大小相关

| 配置项               | 说明                         | 建议                                    |
|-------------------|----------------------------|---------------------------------------|
| `maximumPoolSize` | 最大连接数 默认10                 | 根据并发情况设置 一般为 CPU 核数 × 2               |
| `minimumIdle`     | 最小空闲连接数 默认=maximumPoolSize | 可设为0节省资源 或设为相同保持连接稳定                  |
| `idleTimeout`     | 空闲连接的最大存活时间 默认600000ms     | 如果 minimumIdle 小于 maximumPoolSize 可调低 |
| `maxLifetime`     | 连接最大生命周期 默认1800000ms       | 设置略小于数据库自身连接超时                        |
| `keepaliveTime`   | 保持连接活跃的ping间隔 默认0禁用        | 设置小于数据库空闲超时时间 如600000ms               |

#### 超时设置

| 配置项                 | 说明                  | 建议                 |
|---------------------|---------------------|--------------------|
| `connectionTimeout` | 等待连接的最大时间 默认30000ms | 视业务情况设为1000~5000ms |
| `validationTimeout` | 校验连接的超时时间 默认5000ms  | 可适当调小              |

#### SQL执行相关

| 配置项                    | 说明       | 建议               |
|------------------------|----------|------------------|
| `autoCommit`           | 是否自动提交事务 | 需要手动控制事务时关闭      |
| `readOnly`             | 是否只读连接   | 读写分离 只读操作可设为true |
| `transactionIsolation` | 事务隔离级别   | 可选配置             |

#### 连接检测与泄漏排查

| 配置项                      | 说明                  | 建议                  |
|--------------------------|---------------------|---------------------|
| `connectionTestQuery`    | 连接校验SQL（如 SELECT 1） | 不推荐设置 用 isValid 更高效 |
| `leakDetectionThreshold` | 连接泄漏检测阈值 默认0(关闭)    | 设为业务超时时间如3000ms进行排查 |

---

#### 示例配置代码

``` java
public static void main(String[] args) {

    //新建配置对象
    HikariConfig config = new HikariConfig();
    //基础配置
    config.setJdbcUrl("jdbc:mysql://localhost:3306/test");
    config.setUsername("root");
    config.setPassword("123456");
    
    //各种参数
    config.setMaximumPoolSize(20);
    config.setMinimumIdle(5);
    config.setIdleTimeout(600000);
    config.setMaxLifetime(1800000);
    config.setKeepaliveTime(300000);
    config.setConnectionTimeout(5000);
    
    config.setLeakDetectionThreshold(3000);
    
    //创建数据源
    HikariDataSource ds = new HikariDataSource(config);
    
    //获取连接
    Connection conn = ds.getConnection();
    
    // ... 使用连接
    
    // conn是代理对象 这里的关闭会归还给连接池
    conn.close(); 
}
```

---

### HikariDatasource

Hikari数据源, 内有懒加载, 通常会创建多个数据源, 用于读写分离, 或连接多个数据库.

实现了了DataSource接口, 部分方法未实现.

---

### PoolEntry与ProxyConnection

PoolEntry是池子里的一个条目, 内有底层连接对象, 访问时间, 借出时间, 连接状态, 驱逐标记, 生命周期定时任务, 打开的SQL语句列表, 只读, 自动提交等字段. 使用原子引用更新状态.

ProxyConnection是代理连接, 由ProxyFactory创建, 内有底层连接, 泄漏检测任务, 打开的语句列表, 以及各种配置标记, 调用底层conn的方法时进行其他操作, 进行功能增强, 有dirtyBits的判断, isCommitStateDirty的标记等. 其中close方法会把连接归还到连接池, 以及重置各种状态.

---

### ProxyStatement和ProxyResultSet

指定操作前, 调用代理连接的markCommitStateDirty方法进行标记, 用于在归还连接的时候判断, 如果不是自动提交还脏了, 则需要进行rollback

---

### HikariPool

数据库连接池, 内有大量配置字段, 挂起恢复锁, 清理任务, 各种工厂和任务. 连接条目放在ConcurrentBag内. getConnection方法用于获取连接, 先获取挂起恢复锁, 再从ConcurrentBag借一个连接, metricsTracker记录状态

--- 

### ConcurrentBag

优于LinkedBlockingQueue和LinkedTransferQueue的并发集合, 适用于连接池, 使用ThreadLocal避免锁和AQS, 使用CopyOnWriteArrayList存储所有资源用于共享, 使用SynchronousQueue避免加锁复制和排队, 避免上下文和内存开销, 使用FastList提高访问效率.

borrow时, 先从ThreadLocal的List内获取, 如果没有则遍历共享列表, 如果还没有则从SynchronousQueue获取(同步等待归还连接).

requite时, 如果有等待的则通过SynchronousQueue递交, 否则归还到ThreadLocal的List内.

--- 

### FastList

轻量级列表, 内部用数组实现, 避免了ArrayList的一些边界检查和扩容开销, 适合高频增删, 尤其在连接池这种场景下效率更好.

---

### JavassistProxyFactory

字节码代理工厂, 填充代码用的, 构建时执行, 通过插件或脚本触发. 给方法加上try-catch, 添加checkException检测异常, 内有检测连接失效, 处理链接泄漏, 状态同步等功能. 再把代理工厂内的方法替换成新生成的类. 简化了繁琐且重复的代码.

---

### ProxyLeakTask

泄漏检测任务, 任务触发时, 代表很久没有归还, 则认为是泄漏, 打印日志警告.

---

### HouseKeeper

定期维护连接池状态, 清理超时空闲链接, 调整连接池大小, 检测连接泄漏, 更新时间基准.

---

## 总结

HikariCP 是一个高性能 JDBC 连接池框架 它通过一个池类(HikariPool)统一管理连接对象(PoolEntry)并对原生连接封装为代理连接(ProxyConnection)实现连接复用 事务控制 泄漏检测等功能 框架通过配置类(HikariConfig)集中配置连接参数 采用 Javassist 动态生成代理类提升性能 同时利用后台线程(HouseKeeper)定期清理无效连接 保证连接池健康稳定stList

