# SQL Tracing

For the global services that started by DBPack proxy, the HTTP headers `traceparent` will be automatically injected.  The `traceparent` value format is:

`00-85d85c3112590a76d0723eed4326dbd8-81e51018180f4913-01`

The value format is:

```golang
fmt.Sprintf("%.2x-%s-%s-%s",
		supportedVersion,
		sc.TraceID(),
		sc.SpanID(),
		flags)
```

The `traceparent` contains `TraceID` and `SpanID`, so that user can construct TraceContext in their microservices and pass through the business logic, and eventually we can get an integrated tracing chain of distributed transaction.

You can also pass `traceparent` through SQL scripts that will be proxied by DBPack, so that DBPack can trace the chain of distributed transaction.

For example:

```
update /*+ XID('gs/aggregationSvc/72343404027518979') TraceParent('00-85d85c3112590a76d0723eed4326dbd8-81e51018180f4913-01') */ product.inventory set available_qty = available_qty - ?, allocated_qty = allocated_qty + ? where product_sysno = ? and available_qty >= ?
```

Please refer to this complete sample here: https://github.com/cectc/dbpack-samples/tree/main/go

DBPack provides feature that allows user to export trace result to jaeger. You can just add following configuration to enable it: 

```yaml
trace:
  jaeger_endpoint: http://jaeger:14268/api/traces # please replace to actual jaeger address
```

Following picture shows some example data of the tracing chain of an integrated distributed transaction, you can see what SQL been executed by DBPack on which DB, and the execution time:

<img src="https://cectc.github.io/dbpack-doc/images/image-20220719145659901.png" alt="image-20220719145659901" style="width:1000px" />
