# 分布式事务

> DBPack 分布式事务支持 EAT (Event Driven Automatic Transaction) 模式和 TCC (Try Confirm Cancel) 模式。EAT 模式支持自动生成 SQL 补偿语句，对数据库操作进行回滚；TCC 模式支持对拦截到的 HTTP1  请求自动调用提交、回滚接口协调分布式事务。

## 原理

DBPack EAT 模式和 TCC 模式的分布式解决方案都基于两阶段提交 (2PC: tow phase commit) 理论。分布式事务问题的解决方案有多种，除了本项目中的 EAT 模式、TCC 模式，还有 XA 协议方案、SAGA 模式方案、最终一致性方案等。各公司应该根据自己的技术栈、研发能力、业务场景等情况选择适合自己的解决方案。

### EAT 模式

两阶段提交：

- 一阶段：业务数据和 UndoLog 在同一本地事务中提交。
- 二阶段：
  - 异步提交: 删除UndoLog。
  - 异步回滚：通过一阶段的 UndoLog 进行反向补偿。

### TCC 模式

- 一阶段 prepare 预留资源
- 二阶段 commit 或 rollback。

EAT 采用 UndoLog 自动反向生成 SQL 补偿，而TCC 则属于业务补偿，需要手动补偿业务。



## 性能测试

测试环境：2018 款 mac book pro

测试模式：EAT 模式

seata-golang 性能：

```
Concurrency Level: 3
Time taken for tests: 5.017 seconds
Complete requests: 93
Failed requests: 0
Total transferred: 14787 bytes
HTML transferred: 3348 bytes
Requests per second: 18.54 [#/sec] (mean)
Time per request: 161.836 [ms] (mean)
Time per request: 53.945 [ms] (mean, across all concurrent requests)
Transfer rate: 2.88 [Kbytes/sec] received
```

dbpack 性能：

```
Concurrency Level: 3
Time taken for tests: 5.019 seconds
Complete requests: 141
Failed requests: 0
Total transferred: 27636 bytes
HTML transferred: 5076 bytes
Requests per second: 28.09 [#/sec] (mean)
Time per request: 106.796 [ms] (mean)
Time per request: 35.599 [ms] (mean, across all concurrent requests)
Transfer rate: 5.38 [Kbytes/sec] received
```



## 参考资料

[分布式事务和两阶段提交](https://medium.com/geekculture/distributed-transactions-two-phase-commit-c82752d69324)

[XA 协议](https://pubs.opengroup.org/onlinepubs/009680699/toc.pdf)

[SAGA 理论](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf)

