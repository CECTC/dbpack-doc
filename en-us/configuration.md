# Configuration

> The configuration of DBPack draws on the idea from Kubernetes: everything is resource! The listener is a resource, the data source of the physical DB is a resource, and the filter is also a resource. All the resources are combined by the name index.

## 1、Listener

```yaml
listeners:
  # The type of protocol being listened to
  - protocol_type: http
    # The address and port exposed by the listener
    socket_address:
      address: 0.0.0.0
      port: 13000
    # Listener configuration
    config:
      # The address of the backend service of the http listener proxy
      backend_host: localhost:3000
    filters:
      - httpDtFilter
  # The type of protocol being listened to
  - protocol_type: mysql
    # The address and port exposed by the listener
    socket_address:
      address: 0.0.0.0
      port: 13307
    # Listener configuration
    config:
      # The username and password of the mysql agent, map structure, the client connecting to DBPack uses the username and password configured here
      users:
        dksl: "123456"
      # Version of the backend DB
      server_version: "8.0.27"
    # The sql request received by the mysql listener is handed over to the executor named redirect to execute
    executor: redirect
```

The above code shows two listener configurations supported by DBPack, one is `http` type listener and the other is `mysql` type listener.

## 2、Executor

```yaml
executors:
  - name: redirect1
    # sdb mode stands for single db, in this mode, only distributed transaction problems can be solved
    mode: sdb
    config:
      # In single db mode, the physical data source of the backend is product
      data_source_ref: product
  - name: redirect2
    # The rws mode stands for read write splitting. This mode can solve the problem of distributed transactions and read and write splitting.
    mode: rws
    config:
      # Load balancing algorithm, currently supports Random, RoundRobin, RandomWeight algorithms, the first two algorithms will ignore the weight of the data source configuration
      load_balance_algorithm: RandomWeight
      # array of data sources
      data_sources:
        - name: employees-master
          # The weight of the data source, r means read, w means write, and the following numbers represent the weight
          weight: r0w10
        - name: employees-slave
          weight: r10w0
  - name: redirect3
    # The shd mode stands for sharding, that is, sharding database and sharding table, which can solve the problems of distributed transactions, read-write splitting, sharding database and sharding table
    mode: shd
    config:
      # db group array, a db group represents a set of primary and secondary databases of a sharded database.
      db_groups:
        # The drug database is split into two databases, drug_0 and drug_1.
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
      # logic_tables describes the logic of database sharding        
      logic_tables:
        # drug represents the name of the database before the sharding. After the sharding, it will be distinguished by an underscore and a number after the noun, such as drug_0, drug_1
        - db_name: drug
          # drug_resource is the name of the logical table, which is used in the user business to query, and it will be rewritten to the actual table name during actual execution
          table_name: drug_resource
          # Whether to allow full table scans, that is, whether to allow requests to be executed on all table shards
          allow_full_scan: true
          # sharding rule
          sharding_rule:
            # column represents the sharding key
            column: id
            # Sharding algorithm, currently only the sharding algorithm based on digital modulo is implemented, namely NumberMod
            sharding_algorithm: NumberMod
          # topology is used to describe the topology of the logical table
          topology:
            "0": 0-4
            "1": 5-9          
```

The above code shows three executor configurations of DBPack, which are `sdb`, `rws`, `shd` three modes. In `shd` mode, there are two sharded databases `drug_0` and `drug_1` indicating that the `drug` database is split into 2 databases. The `topology` in the above code indicates that the `drug_resource` table is split into 10 sharded tables, which belong to two databases, namely:

```
drug_0: drug_resource_0、drug_resource_1、drug_resource_2、drug_resource_3、drug_resource_4
drug_1: drug_resource_5、drug_resource_6、drug_resource_7、drug_resource_8、drug_resource_9
```

After calculating which shard table needs to execute the sql request based on the shard key, it can be determined which physical DB should be executed according to the topology.

## 3、Filter

```yaml
filters:
  # HttpDistributedTransaction filter, the filter intercepts http requests and handles distributed transaction related logic.
  - name: httpDTFilter
    kind: HttpDistributedTransaction
    conf:
      # The appid of the proxied application
      appid: aggregationSvc
      # transaction_infos If the url of the http request matches the request_path, the interceptor executes the logic of creating a global transaction and injects x-dbpack-xid into the request header.
      transaction_infos:
        - request_path: "/v1/order/create"
          # Global transaction timeout, in milliseconds
          timeout: 60000
          # Exact match, supports [prefix] prefix match, [regex] regular match, default exact
          match_type: exact
        - request_path: "/v1/pay/"
          # Global transaction timeout, in milliseconds
          timeout: 60000
          # prefix match
          match_type: prefix
        - request_path: "/v1/account(/.*)?"
          # Global transaction timeout, in milliseconds
          timeout: 60000
          # regular match
          match_type: regex
  # MysqlDistributedTransaction filter, the filter intercepts mysql requests and handles distributed transaction related logic
  - name: mysqlDTFilter
    kind: MysqlDistributedTransaction
    conf:
      # The appid of the proxied application
      appid: productSvc
      # Global lock query interval
      lock_retry_interval: 100ms
      # Global lock retries
      lock_retry_times: 15
  # ConnectionMetricFilter collect sql execution metrics on connections
  - name: metricFilter
    kind: ConnectionMetricFilter
```

The above code shows the three filter configurations that DBPack currently supports. If you use the distributed transaction function, you need to configure the `HttpDistributedTransaction` and `MysqlDistributedTransaction` filters. filter is referenced by Listener and Executor by name.

## 4、DataSource

```yaml
data_source_cluster:
  # data source name
  - name: employees-master
    # The number of connections kept by the connection pool by default
    capacity: 10
    # The maximum number of connections in the connection pool
    max_capacity: 20
    # The time when the connection idle is recycled
    idle_timeout: 60s
    # connection string
    dsn: root:123456@tcp(dbpack-mysql1:3306)/employees?timeout=1s&readTimeout=1s&writeTimeout=1s&parseTime=true&loc=Local&charset=utf8mb4,utf8
    # Interval to ping physical DB
    ping_interval: 20s
    # If the ping fails three times in a row, the status of the data source is changed to Unknown, and the request traffic will not be obtained. If the ping succeeds three times in a row, the recovery status will be Running.
    ping_times_for_change_status: 3
    # filter on connection
    filters:
      # Referencing filter resources by name
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

The above code shows the configuration of the data source.

## 5、Distributed transaction

```yaml
distributed_transaction:
  appid: svc
  // The maximum retry time for rollback, in milliseconds, that is, if the time specified by this setting is exceeded, the corresponding transaction branch will not be rolled back. The time can be dynamically adjusted according to the characteristics of the business itself.
  retry_dead_threshold: 130000
  // Whether to allow the global lock to be released after the rollback timeout, if the corresponding resource data has been locked, and then modifying the data is not allowed, it is recommended to set it to true, that is, to release the global lock after the rollback timeout.
  rollback_retry_timeout_unlock_enable: true
  // etcd configuration
  etcd_config:
    endpoints:
      - etcd:2379
```

The above code shows the configuration of distributed transactions.