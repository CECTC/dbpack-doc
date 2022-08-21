# Rate Limiting & Circuit Breaker


DBPack supports setting rules of rate limiting and circuit breaker for SQL requests.

## Rate Limiting

Users can enable rate limiting function by setting filter configuration. To set the rule, the first step is to define the `RateLimitFilter` configuration:

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

Users can set different rate limiting for different type of SQL (insert, delete, update, select). The limit rate unit is `request / per second`, indicating how many requests per second is allowed. In upper example, it limits 1000 requests per second.

After defining `RateLimitFilter`, you need to add filter's name to the filter list of Executor:

```yaml
    executors:
      - name: redirect
        mode: sdb
        config:
          data_source_ref: employees
        filters:
          - cryptoFilter
          # rate limiting filter
          - rateLimiterFilter
```

That's it! The rate limit configuration is all set!

## Circuit Breaker

To configure circuit breaker function of DBPack, we need to define `CircuitBreakerFilter` first: 

```yaml
      - name: circuitBreakerFilter
        kind: CircuitBreakerFilter
        conf:
          # error count
          error_threshold: 20
          # success count
          success_threshold: 5
          # seconds
          timeout: 60
```

Above configuration indicates:

1. If error count reaches 20 in 60 seconds, the circuit breaker status will be switched to `Open` and requests can not be executed for a while.
2. After the circuit breaker switched to `Open` status for 60 seconds, its status will change to `HalfOpen`, at this time, requests can be executed.
3. When the circuit breaker status been switched to `HalfOpen`, if the first coming request execution result is failed, the break will switch to `Open` again; On the other hand, if 5 consecutive requests are executed successfully, the breaker will be turned off, and the fuse status will be switched to `Closed`.

After defining `CircuitBreakerFilter`, we need to add the filter name to the filter list of Executor

```yaml
    executors:
      - name: redirect
        mode: sdb
        config:
          data_source_ref: employees
        filters:
          - cryptoFilter
          # circuit break filter
          - circuitBreakerFilter
```

All done! That's all for the circuit breaker configuration.
