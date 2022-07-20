# Health Status

> DBPack provides `/live` API to check its life status, and `/ready` API to check if it's ready to serve. When all the `DataSource` can be successfully pinged, `/ready` API response HTTP status code 200. DBPack also provides `/status` API to check DBPack status, and `/metrics` API to check DBPack running metrics.

+ `/status` API

  Response example:

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

  This API reflects if the DBPack configuration `Listener` is `active`, whether to enable distributed transaction. It also shows if multiple DBPack replicas are `master` or not.

+ `/metrics` API

  This API responses metrics that the DBPack collects.

Above APIs provide service via `http_listen_port` configuration. The default port is 18888, for example, to access upper API, here is the integral URL example:

```
http://${ip}:18888/live
http://${ip}:18888/ready
http://${ip}:18888/status
http://${ip}:18888/metrics
```