# 分布式事务

> DBPack 分布式事务支持 AT 模式和 TCC 模式。AT 模式支持自动生成 SQL 补偿语句，对数据库操作进行回滚；TCC 模式支持对拦截到的 HTTP1  请求自动调用提交、回滚接口协调分布式事务。

## 原理

DBPack AT 模式和 TCC 模式的分布式解决方案都基于两阶段提交 (2PC: tow phase commit) 理论。分布式事务问题的解决方案有多种，除了本项目中的 AT 模式、TCC 模式，还有 XA 协议方案、SAGA 模式方案、最终一致性方案等。各公司应该根据自己的技术栈、研发能力、业务场景等情况选择适合自己的解决方案。

## 参考资料

[分布式事务和两阶段提交](https://medium.com/geekculture/distributed-transactions-two-phase-commit-c82752d69324)

[XA 协议](https://pubs.opengroup.org/onlinepubs/009680699/toc.pdf)

[SAGA 理论](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf)