# 限流熔断

DBPack 支持对 SQL 请求设置限流熔断规则。

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