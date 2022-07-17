# Sharding Databases && Tables

> Currently, DBPack supports cross-shard query, cross-DB query, Order By, and Limit.

There are integrated testing case in the dbpack repository, which can be verified by running [docker-compose](https://github.com/dk-lockdown/dbpack/blob/dev/docker/docker-compose-shd.yaml) to set up a testing environment quickly. The test case uses 2 databases and 10 tables. Each DB has 5 sharding tables. You can check sharding details from this [sql script](https://github.com/dk-lockdown/dbpack/tree/dev/docker/scripts).

After the testing environment is ready, you can run [sharding_test.go](https://github.com/dk-lockdown/dbpack/blob/dev/test/shd/sharding_test.go) to check the integration test result.

You can also test by connecting to the mysql server provided by DBPack proxy:

```
mysql -h 127.0.0.1 -P 13306 -u dksl -p123456

> select id, drug_res_type_id, base_type from drug_resource where id between 200 and 210;

> select id, drug_res_type_id, base_type from drug_resource where id between 200 and 210 order by id desc ;

> select id, drug_res_type_id, base_type from drug_resource where id between 200 and 210 order by id desc limit 10, 30;

> select id, drug_res_type_id, base_type from drug_resource where id between 200 and 210 order by id desc limit 30;
```

