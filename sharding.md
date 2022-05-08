# 分库分表

> DBPack 当前支持跨分片查询、跨 DB 查询、order by、limit。

仓库当中有集成测试用例，可通过运行用例中的 [docker-compose 文件](https://github.com/dk-lockdown/dbpack/blob/dev/docker/docker-compose-shd.yaml)来快速启动测试验证环境。该用例构造了一个 2 库 10 表的逻辑分片表，每个 DB 5 个分片表，具体分片情况请查看 [sql 脚本](https://github.com/dk-lockdown/dbpack/tree/dev/docker/scripts)。

测试环境启动后，可运行 [sharding_test.go](https://github.com/dk-lockdown/dbpack/blob/dev/test/shd/sharding_test.go) 集成测试用例进行测试。

也可通过 mysql 客户端连接 DBPack 测试，例如：

```
mysql -h 127.0.0.1 -P 13306 -u dksl -p123456

> select id, drug_res_type_id, base_type from drug_resource where id between 200 and 210;

> select id, drug_res_type_id, base_type from drug_resource where id between 200 and 210 order by id desc ;

> select id, drug_res_type_id, base_type from drug_resource where id between 200 and 210 order by id desc limit 10, 30;

> select id, drug_res_type_id, base_type from drug_resource where id between 200 and 210 order by id desc limit 30;
```

