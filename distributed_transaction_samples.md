# samples

> DBPack 支持任意编程语言，目前仓库中有 go、java 语言的例子，php 和 python 的例子正在制作中。

### 第一步：clone 代码

```
git clone git@github.com:CECTC/dbpack.git
cd dbpack
```

### 第二步：安装 ETCD

### 第三步：安装数据库，使用下面的脚本初始化数据库
```
./samples/go/scripts/order.sql
./samples/go/scripts/product.sql
```
### 第四步：编译 dbpack，启动 dbpack 代理

```bash
make build

vim ./samples/go/config1.yaml
# 修改配置 distributed_transaction.etcd_config.endpoints

vim ./samples/go/config2.yaml
# 修改配置 data_source_cluster.dsn
# 修改配置 distributed_transaction.etcd_config.endpoints

vim ./samples/go/config3.yaml
# 修改配置 data_source_cluster.dsn
# 修改配置 distributed_transaction.etcd_config.endpoints

./dist/dbpack start --config ./samples/go/config1.yml

./dist/dbpack start --config ./samples/go/config2.yml

./dist/dbpack start --config ./samples/go/config3.yml
```

### 第五步：运行 aggregation_svc client
```bash
cd samples/go/

go run aggregation_svc/main.go
```

### 第六步：运行 order_svc client
```bash
cd samples/go/
vim ./order_svc/main.go
# 修改配置 dsn

go run order_svc/main.go
```

### 第七步：运行 product_svc client
```bash
cd samples/go/
vim ./product_svc/main.go
# 修改配置 dsn

go run product_svc/main.go
```

### 第八步：访问测试
```
curl -XPOST http://localhost:13000/v1/order/create
```