# DBPack

## [中文](/README.md) | English

<img src="https://cectc.github.io/dbpack-doc/images/dbpack.svg" alt="image-20220427091848831" />

> It is a database proxy that aims to solve the distributed transaction problems during business development, which provides solutions for read && write splitting, database and table sharding. Through the mesh-based deployment method, DBPack shields complex basic logic, so that business development does not need to rely on a specific SDK, which simplifies the development process and improves development efficiency.

## Features

+ support MYSQL protocol.
+ event driven distributed transaction solution inspired by kubernetes.
+ support read && write splitting, customized SQL route through `Hint`
+ support audit log, support sql tracing.
+ sharding: support query on sharded table, support order by，support limit.
+ sharding: support insert、delete、update on sharded table
+ can be deployed as a sidecar, so that any programming language can use it to handle distributed transaction
+ more features on the road

## Prerequisites

+ Go >= 1.17
+ MYSQL >= 5.7

## Architecture

![architecture](../images/arch-for-dbpack.drawio.png)

+ Listener：parse SQL protocol
+ Executor：forward SQL request to actual DB server
+ Filter：quota statistics, SQL interception, sensitive information encryption and decryption, etc
+ ConnectionFilter：handle SQL interception on the DB connection

## Discussion Group

Please scan below OR code through WeChat and reply "join group".

<img src="https://cectc.github.io/dbpack-doc/images/image-20220427091848831.png" alt="image-20220427091848831" style="zoom:50%" align="left"/>
