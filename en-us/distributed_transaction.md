# Distributed Transaction

> DBPack supports both EAT(Event Driven Automatic Transaction) and TCC(Try Confirm Cancel) mode. For EAT mode, it will generate compensated SQL and automatically rollback the branch transaction in case global transaction rollback. For TCC mode, it will intercept HTTP1 request and coordinate distributed transaction by calling commit or rollback API automatically.

## Theory

Both the EAT and TCC distributed transaction are based on 2PC (tow phase commit) theory. There are many solutions for distributed transaction, besides EAT mode, TCC mode, there are XA mode, SAGA mode, eventual consistency solution and so on. Companies should select suitable solutions according to their technic stack, development capacity, business scenario...etc.



## Performance Testing

Testing Environment: MacBook Pro 2018

Testing mode: AT

Seata-golang Performance:

```
Concurrency Level: 3
Time taken for tests: 5.017 seconds
Complete requests: 93
Failed requests: 0
Total transferred: 14787 bytes
HTML transferred: 3348 bytes
Requests per second: 18.54 [#/sec] (mean)
Time per request: 161.836 [ms] (mean)
Time per request: 53.945 [ms] (mean, across all concurrent requests)
Transfer rate: 2.88 [Kbytes/sec] received
```

DBPack Performance:

```
Concurrency Level: 3
Time taken for tests: 5.019 seconds
Complete requests: 141
Failed requests: 0
Total transferred: 27636 bytes
HTML transferred: 5076 bytes
Requests per second: 28.09 [#/sec] (mean)
Time per request: 106.796 [ms] (mean)
Time per request: 35.599 [ms] (mean, across all concurrent requests)
Transfer rate: 5.38 [Kbytes/sec] received
```


## References

[Distributed Transactions and Two-Phase Commit](https://medium.com/geekculture/distributed-transactions-two-phase-commit-c82752d69324)

[XA Protocol](https://pubs.opengroup.org/onlinepubs/009680699/toc.pdf)

[SAGA Theory](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf)

