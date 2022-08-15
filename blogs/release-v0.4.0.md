# DBPack 限流熔断功能发布说明

上周我们发布了 v0.4.0 版本，增加了限流熔断功能，现对这两个功能做如下说明。

## 限流

DBPack 限流熔断功能通过 filter 实现。要设置限流规则，首先要定义 `RateLimitFilter`：

```yaml
    - name: rateLimiterFilter
      kind: RateLimiterFilter
        conf:
        # 1000 requests per second
        insert_limit: 1000
        # 1000 requests per second
        update_limit: 1000
        # 1000 requests per second
        delete_limit: 1000
        # 1000 requests per second
        select_limit: 1000
```

支持对增删改查请求单独限流。限流策略以秒为单位，即允许每秒执行多少次请求（RPS）。如果设置为 0 表示不限流。上面的例子表示限制为每秒执行 1000 次请求。

定义好 `RateLimitFilter` 后，将 filter 的名字加入到 Executor 的 filter list 中：

```yaml
    executors:
      - name: redirect
        mode: sdb
        config:
          data_source_ref: employees
        filters:
          - cryptoFilter
          # 限流 filter
          - rateLimiterFilter
```

这样就配置好限流功能了。

## 熔断

配置 DBPack 的熔断功能，需要先定义 `CircuitBreakerFilter`：

```yaml
      - name: circuitBreakerFilter
        kind: CircuitBreakerFilter
        conf:
          # error 次数
          error_threshold: 20
          # success 次数
          success_threshold: 5
          // seconds
          timeout: 60
```

上面的配置表示：

1. 60 秒内累计错误次数达到 20 次，熔断器状态为 `Open` 打开状态，此时请求不能执行。
2. 熔断器打开 60 秒后，熔断器状态变为 `HalfOpen` 半开状态，此时可以执行请求。
3. 熔断器状态变为 `HalfOpen` 半开状态后，执行的第一个请求，如果执行失败，熔断器再次变为 `Open` 打开状态；如果连续 5 次请求执行成功，则关闭熔断器，熔断器状态变为 `Closed`。

定义好 `CircuitBreakerFilter` 后，将 filter 的名字加入到 Executor 的 filter list 中：

```yaml
    executors:
      - name: redirect
        mode: sdb
        config:
          data_source_ref: employees
        filters:
          - cryptoFilter
          # 熔断 filter
          - circuitBreakerFilter
```

这样就配置好了熔断功能。

## 综述

在 v0.1.0 版本我们发布了分布式事务功能，支持各种编程语言协调分布式事务。

在 v0.2.0 版本我们发布了读写分离功能，用户在开启读写分离功能的情况下，使用分布式事务协调功能不再需要做复杂的集成，DBPack 提供了一站式的解决方案。

在 v0.3.0 版本，我们加入 SQL Tracing 的功能，使用该功能可以收集到一个完整的分布式事务链路，查看事务的执行情况。我们还加入了数据加密功能，通过该功能保护用户的重要数据资产。

在 v0.4.0 版本，我们加入了限流熔断功能，该功能能保护数据库不受到超过自身处理能力的请求流量冲击。

在 v0.5.0 版本中，我们将加入分库分表功能。

欢迎开源爱好者和我们一起建设 DBPack 社区，加群或参与社区建设，请微信联系：scottlewis。

## 链接

- dbpack: https://github.com/CECTC/dbpack
- dbpack-samples: https://github.com/CECTC/dbpack-samples
- dbpack-doc: https://github.com/CECTC/dbpack-doc
- 事件驱动分布式事务设计：https://mp.weixin.qq.com/s/r43JvRY3LCETMoZjrdNxXA
- 视频介绍：
  - 《dbpack 分布式事务功能详解》 https://www.bilibili.com/video/BV1cg411X7Ek
  - 《高性能分布式事务框架实践》https://www.bilibili.com/video/BV1Xr4y1L7kD