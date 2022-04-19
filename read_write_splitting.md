# 读写分离

+ DBPack 能根据 SQL 操作的类型，自动路由 `Insert`、`Update`、`Delete` 请求到主库，`Select` 请求到从库。

+ DBPack 还能通过 Hint `UseDB('%s')` 自定义查询请求路由。

+ 在开启事务的情况下，所有请求都将路由到主库。

+ DBPack 支持通过如下策略剔除无法访问的从库：

  ```
      ping_interval: 20s
      ping_times_for_change_status: 3
  ```

  通过设置 `ping` 数据库的间隔，以及 `ping` 的次数，来改变 DB 的状态。在上面的例子中，如果连续三次 `ping` 失败，则 DB 的状态改为 `Unkown`，如果连续三次 `ping` 成功，则 DB 的状态改为 `Running`。`Unknown` 状态的 DB 将不会获得访问流量。