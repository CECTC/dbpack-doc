# DBPack v0.1.0 发布公告

**经过一个多月的努力，DBPack 发布了今年第一个版本，该版本 Release 了分布式事务解决方案，并提供读写分离功能的预览。DBPack 支持任何微服务编程语言，我们已经准备了 [go](https://github.com/CECTC/dbpack-samples/blob/main/go/README.md)、[java](https://github.com/CECTC/dbpack-samples/blob/main/java/README.md)、[php](https://github.com/CECTC/dbpack-samples/blob/main/php/README.md)、[python](https://github.com/CECTC/dbpack-samples/blob/main/python/README.md) 的示例。**

### 下面是我们修复的 Bug：

- 增加延迟退出配置 ([#4](https://github.com/cectc/dbpack/issues/4)) ([6604ce8](https://github.com/cectc/dbpack/commit/5c607d48d1149218cff3988dcb00d83da571a561))
- 使用 `db` 代理 `tx` 执行 sql  ([#8](https://github.com/cectc/dbpack/pull/8)) ([7e2b42d](https://github.com/cectc/dbpack/commit/52d78cab0bc414d92a5c59230f2827c8332c2bde))
- 当收到 ComQuit 请求，应该归还连接 ([#51](https://github.com/cectc/dbpack/pull/51)) ([627adc2](https://github.com/cectc/dbpack/commit/e8f07086ccf76a7112f00512e3ed3f6e94aff410))
- 从连接上读取 sql 执行结果完毕应该关闭 `statement` ([#71](https://github.com/cectc/dbpack/pull/71)) ([f924e10](https://github.com/cectc/dbpack/commit/4c9a29271d73df0ff8daf92c3faebf1540b0cf01))
- `ping` 数据库后应该归还连接 ([#74](https://github.com/cectc/dbpack/pull/74)) ([07de56e](https://github.com/cectc/dbpack/commit/c1c77710398ad58d7d3809ad66312550b0931236))
- 处理超时的全局事务应该刷新全局事务的状态 ([#86](https://github.com/cectc/dbpack/pull/86)) ([3046e17](https://github.com/cectc/dbpack/commit/6bf4090fbe897c60c229ac172fdb0c14720066ee))
- 当 `undologs` 不存在的时候应该释放 `tx` 对象  ([#93](https://github.com/cectc/dbpack/pull/93)) ([7aeaa4e](https://github.com/cectc/dbpack/commit/99df0361ca1cf7876daa66151cf6bb462d0fd3bb))

### 下面是一些重大的特性：

- etcd watch 机制驱动的分布式事务 ([#11](https://github.com/cectc/dbpack/pull/11)) ([ce10990](https://github.com/cectc/dbpack/commit/e9910501e32d23741f99f5fe9ece1077ba1b348c))
- 支持 TCC 模式事务分支提交回滚 ([#12](https://github.com/cectc/dbpack/issues/12)) ([c0bfdf9](https://github.com/cectc/dbpack/commit/feab7aefe819bf3217363994c67515b887f8adb9))
- 支持 `GlobalLock` Hint ([#14](https://github.com/cectc/dbpack/issues/14)) ([8369f8f](https://github.com/cectc/dbpack/commit/5c7c96797539943ed75495d1cfa92f6094ff548e))
- 支持 Leader 选举，只有 Leader 可以处理事务的提交回滚 ([#19](https://github.com/cectc/dbpack/pull/19)) ([b89c672](https://github.com/cectc/dbpack/commit/d7ab60b6ed5547f1bc9a6c426e1fb9ee21d6f4f3))
- 增加 Prometheus 指标 ([#25](https://github.com/cectc/dbpack/issues/25)) ([627adc2](https://github.com/cectc/dbpack/commit/627adc2ced9da499e6b658f718b23417e7df9903))
- 增加 `readiness` 和 `liveness` 探针 ([#52](https://github.com/cectc/dbpack/issues/52)) ([fd889cc](https://github.com/cectc/dbpack/commit/f43ab5f4ed6eafaf950a73e241c536849a16e4f9))

### 一些修改：

- 优化分支事务处理逻辑 ([#17](https://github.com/cectc/dbpack/pull/17)) ([c6a6626](https://github.com/cectc/dbpack/commit/06d624511c65a379e73dae91c2be4fb3785b9bf0))

### 下面是为本次版本发布做出贡献的贡献者名单，非常感谢大家的付出：

- [@rocymp](https://github.com/rocymp) 
- [@gorexlv](https://github.com/gorexlv) 
- [@zackzhangkai](https://github.com/zackzhangkai) 
- [@yx9o](https://github.com/yx9o) 
- [@bohehe](https://github.com/bohehe) 
- [@fatelei](https://github.com/fatelei) 
- [@zhu733756](https://github.com/zhu733756) 
- [@wybrobin](https://github.com/wybrobin) 
- [@tanryberdi](https://github.com/tanryberdi) 
- [@JuwanXu](https://github.com/JuwanXu) 
- [@hzliangbin](https://github.com/hzliangbin) 