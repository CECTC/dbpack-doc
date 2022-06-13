# 配置说明

> DBPack 的配置吸取了 kubernetes 的思想，everything is resource！listener 是资源，物理 DB 的 data source 是资源，filter 也是资源，所有的资源通过 name 索引组合起来。

## 一、Listener 监听器

```yaml
listeners:
  # 监听的协议类型
  - protocol_type: http
    # 监听器暴露的地址和端口
    socket_address:
      address: 0.0.0.0
      port: 13000
    # 监听器的配置
    config:
      # http 监听器代理的后端服务的地址
      backend_host: localhost:3000
    filters:
      - httpDtFilter
  # 监听的协议类型
  - protocol_type: mysql
    # 监听器暴露的地址和端口
    socket_address:
      address: 0.0.0.0
      port: 13307
    # 监听器的配置
    config:
      # mysql 代理的用户名和密码，map 结构，连接 DBPack 的客户段使用这里配置的用户名和密码
      users:
        dksl: "123456"
      # DBPack 代理的后端 DB 的版本
      server_version: "8.0.27"
    # mysql 监听器接收到的 sql 请求交给名为 redirect 的 executor 去执行
    executor: redirect
```

上面的代码展示了 DBPack 支持的两种监听器配置，一种是 `http` 类型的监听器，一种是 `mysql` 类型的监听器。

## 二、Executor 执行器

```yaml
executors:
  - name: redirect1
    # sdb 模式代表 single db，在该模式下，仅能解决分布式事务问题
    mode: sdb
    config:
      # single db 模式下，后端的物理数据源为 product
      data_source_ref: product
  - name: redirect2
    # rws 模式代表 read write splitting，即读写分离，该模式下可以解决分布式事务问题和读写分离问题
    mode: rws
    config:
      # 负载均衡算法，当前支持 Random、RoundRobin、RandomWeight 算法，前两种算法会无视 data source 配置的权重
      load_balance_algorithm: RandomWeight
      # 数据源数组
      data_sources:
        - name: employees-master
          # 数据源权重，r 表示读，w 表示写，后面的数字表示权重
          weight: r0w10
        - name: employees-slave
          weight: r10w0
  - name: redirect3
    # shd 模式代表 sharding，即分库分表，该模式下可解决分布式事务、读写分离、分库分表问题
    mode: shd
    config:
      # db group 数组，一个 db group 表示一个分库的主备库集合
      db_groups:
        # drug 数据库被拆分成了两个数据库，分别是 drug_0、drug_1。
        - name: drug_0
          load_balance_algorithm: RandomWeight
          data_sources:
            - name: drug_0
              weight: r10w10
        - name: drug_1
          load_balance_algorithm: RandomWeight
          data_sources:
            - name: drug_1
              weight: r10w10
      # logic_tables 描述了数据分片的逻辑        
      logic_tables:
        # drug 表示分库前的数据库名称，分库后，会在名词后面加上下划线和数字来区分，比如 drug_0、drug_1
        - db_name: drug
          # drug_resource 为逻辑表名称，用户业务中使用该表名查询，实际执行时会被改写为真正执行的表名
          table_name: drug_resource
          # 是否允许全表扫描，即是否允许请求在所有表分片上执行
          allow_full_scan: true
          # sharding 规则
          sharding_rule:
            # column 表示分片键
            column: id
            # 分片算法，当前只实现了基于数字取余的分片算法，即 NumberMod
            sharding_algorithm: NumberMod
          # topology 用于描述逻辑表的拓扑结构
          topology:
            "0": 0-4
            "1": 5-9          
```

上面的代码展示了 DBPack 的三种执行器配置，它们是 `sdb`、`rws`、`shd` 三种模式。在 `shd` 模式下，存在两个分库 `drug_0` 和 `drug_1` 表示 `drug` 这个数据库被拆分成了 2 个数据库。上面代码的中的 `topology` 表示 `drug_resource` 这张表被拆分成了 10 张分片表，他们分属两个数据库，即：

```
drug_0: drug_resource_0、drug_resource_1、drug_resource_2、drug_resource_3、drug_resource_4
drug_1: drug_resource_5、drug_resource_6、drug_resource_7、drug_resource_8、drug_resource_9
```

根据分片键计算出需要在哪个分片表上执行 sql 请求后，根据拓扑结构就能得出应该在哪个物理 DB 上去执行。

## 三、Filter 过滤器

```yaml
filters:
  # HttpDistributedTransaction filter，该 filter 拦截 http 请求，处理分布式事务相关逻辑
  - name: httpDTFilter
    kind: HttpDistributedTransaction
    conf:
      # 被代理的应用的 appid
      appid: aggregationSvc
      # transaction_infos 如果 http 请求的 url 和 request_path 匹配，则拦截器会执行创建全局事务的逻辑，并将 x_dbpack_xid 注入请求 header 中。
      transaction_infos:
        - request_path: "/v1/order/create"
          # 全局事务超时时间，单位毫秒
          timeout: 60000
          // 完全匹配, 支持 [prefix] 前缀匹配、[regex] 正则匹配，默认 exact
          match_type: exact
        - request_path: "/v1/pay/"
          # 全局事务超时时间，单位毫秒
          timeout: 60000
          // 前缀匹配
          match_type: prefix
        - request_path: "/v1/account(/.*)?"
          # 全局事务超时时间，单位毫秒
          timeout: 60000
          // 正则匹配
          match_type: regex
  # MysqlDistributedTransaction filter，该 filter 拦截 mysql 请求，处理分布式事务相关逻辑
  - name: mysqlDTFilter
    kind: MysqlDistributedTransaction
    conf:
      # 被代理的应用的 appid
      appid: productSvc
      # 全局锁查询间隔
      lock_retry_interval: 100ms
      # 全局锁重试次数
      lock_retry_times: 15
  # ConnectionMetricFilter 收集连接上的 sql 执行指标
  - name: metricFilter
    kind: ConnectionMetricFilter
```

上面的代码展示了 DBPack 现在支持的三种 filter 的配置。如果使用分布式事务功能，需配置 `HttpDistributedTransaction` 和 `MysqlDistributedTransaction` filter。filter 通过名称被 Listener 和 Executor 引用。

## 四、DataSource 配置

```yaml
data_source_cluster:
  # 数据源名称
  - name: employees-master
    # 连接池默认保持的连接数
    capacity: 10
    # 连接池最大连接数
    max_capacity: 20
    # 连接空闲被回收的时间
    idle_timeout: 60s
    # 连接字符串
    dsn: root:123456@tcp(dbpack-mysql1:3306)/employees?timeout=1s&readTimeout=1s&writeTimeout=1s&parseTime=true&loc=Local&charset=utf8mb4,utf8
    # ping 物理 DB 的间隔
    ping_interval: 20s
    # 如果连续三次 ping 失败，数据源的状态改为 Unknown，将不会获得请求流量，如果连续 ping 三次成功，则恢复状态为 Running。
    ping_times_for_change_status: 3
    # 连接上的 filter
    filters:
      # 通过名称引用 filter 资源
      - mysqlDTFilter

  - name: employees-slave
    capacity: 10
    max_capacity: 20
    idle_timeout: 60s
    dsn: root:123456@tcp(dbpack-mysql2:3306)/employees?timeout=60s&readTimeout=60s&writeTimeout=60s&parseTime=true&loc=Local&charset=utf8mb4,utf8
    ping_interval: 20s
    ping_times_for_change_status: 3
    filters:
      - mysqlDTFilter
```

上面的代码展示了数据源的配置。

## 五、分布式事务

```yaml
distributed_transaction:
  appid: svc
  // 回滚最大重试时间，单位毫秒，即超过该设置声明的时间，相应的事务分支不再回滚，该时间可根据业务自身的特性动态调整。
  retry_dead_threshold: 130000
  // 是否允许回滚超时后释放全局锁，如果相应资源数据一直被锁住，再修改该数据则不被允许，建议设置为 true，即回滚超时后释放全局锁。
  rollback_retry_timeout_unlock_enable: true
  // etcd 配置
  etcd_config:
    endpoints:
      - etcd:2379
```

上面的代码展示了分布式事务的配置。