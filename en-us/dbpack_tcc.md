# TCC Mode

<img src="https://cectc.github.io/dbpack-doc/images/distributed-transaction-en.gif" alt="image-20220427100734991" style="zoom:67%;" />

+ When sending request to the aggregation service, the DBPack sidecar will intercept the request, and generate `GlobalTransaction` data and global unique mark `XID`.

***

+ When calling order service, the `XID` will be passed to order service context via HTTP Header.
+ The request will be intercepted by sidecar, and DBPack will check if there is `XID` in the HTTP Headers. Then the request body, as the carried data of `BranchTransaction`, will be serialized and stored in ETCD.

***

+ When calling order service, the `XID` will be passed to order service context via HTTP Header.
+ The request will be intercepted by sidecar, and DBPack will check if there is `XID` in the HTTP Headers. Then the request body, as the carried data of `BranchTransaction`, will be serialized and stored in ETCD.

***

+ Back to the aggregation service, after the business logic been executed, if there is no exception nor error, DBPack will get `BranchTransaction` through `XID`, then it will send `Commit` request using the data from request body. Otherwise, it will send `Rollback` request.
