# 健康状态

> DBPack 支持 `/live` 接口检查存活状态，`/ready` 接口检查就绪状态，当 DBPack 对接的所有 `DataSource` 能 ping 通时 `/ready` 接口返回 200 状态码。DBPack 还支持 `/status` 状态接口，查看 DBPack 状态，以及 `/metrics` 接口查看 DBPack 运行指标。

+ `/status` 接口

  返回内容示例：

  ```json
  {
  	"listeners": [{
  		"protocol_type": "mysql",
  		"socket_address": {
  			"address": "0.0.0.0",
  			"port": 13306
  		},
  		"active": true
  	}],
  	"distributed_transaction_enabled": true,
  	"is_master": true
  }
  ```

  该接口反馈 DBPack 配置的 `Listener` 是否 `active`，是否开启分布式事务功能，以及多个副本的 DBPack 是否是 `master`。

+ `/metrics` 接口

  该接口反馈 DBPack 运行时收集到的指标。

上述接口通过 `http_listen_port` 暴露出来对外提供服务，默认端口是 18888。以默认端口为例，访问上述接口时，完整的 URL 如下所示：

```
http://${ip}:18888/live
http://${ip}:18888/ready
http://${ip}:18888/status
http://${ip}:18888/metrics
```

