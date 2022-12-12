# 分库分表

> DBPack 当前支持跨分片查询、跨 DB 查询、Order By、Limit、自动生成主键、影子表等功能，下面介绍一下技术细节。

## 优化器

进入 DBPack 的 SQL 请求，首先会被 SQL Parser 转换为 AST 抽象语法树，然后经过优化器生成执行计划。当前优化器支持如下逻辑：

### WHERE Condition 预处理

1. 函数提前计算

- SELECT 1 + 1 ,  优化为SELECT 2

- SELECT 1 = 1 ,  优化为SELECT 1

- SELECT 0 = 1 ,  优化为SELECT 0 

- 支持下列字符函数计算：

  ```LENGTH、CHAR_LENGTH、CHARACTER_LENGTH、CONCAT、CONCAT_WS、LEFT、LOWER、LPAD、LTRIM、REPEAT、REPLACE、REVERSE、RIGHT、RPAD、RTRIM、SPACE、STRCMP、SUBSTRING、UPPER```

- 支持下列数学函数计算：

  ```ABS、CELL、EXP、FLOOR、MOD、PI、POW、POWER、RAND、ROUND、SIGN、SQRT```

- 支持下列日期函数计算：

  ```ADDDATE、CURDATE、CURRENT_DATE、CURRENT_TIMESTAMP、CURTIME、DATEDIFF、DATE_FORMAT、DAY、DAYNAME、DAYOFMONTH、DAYOFWEEK、DAYOFYEAR、FROM_UNIXTIME、HOUR、LOCALTIME、MINUTE、MONTHNAME、MONTH、NOW、QUARTER、SECOND、SUBDATE、SYSDATE、UNIX_TIMESTAMP```

- 支持下列 CAST 函数：

  ```CAST、CONVERT```

- 其他函数：

  ```IF、IFNULL、MD5、SHA、SHA1```

计算分片时，可根据这些函数返回值进行计算。例如：

```
SELECT film_id, title, rental_rate, rental_duration FROM film WHERE rental_duration = POW(2,2)
```

如果以 `rental_duration` 余 4 分片，则可得分片结果为 0，则可将查询条件改写为：

```
SELECT film_id, title, rental_rate, rental_duration FROM film_0 WHERE rental_duration = POW(2,2)
```

执行上面的查询语句可得到正确的查询结果。

2. 判断永真/永假式，短化路径

比如:

- false or a = 1，优化为 a = 1
- true or a = 1，优化为 TrueCondition
- false and a =1 , 优化为 FalseCondition
- true and a = 1 , 优化为 a = 1
- 1  and a = 1， 优化为 a = 1  (非 0 的常量值代表 true，0 值代表为 false)

**TrueCondition 意味着全表扫描；FalseCondition 意味着 SQL 请求未落在任何分片，查询结果为空。**

3. 智能合并 and/or 的范围查询

几种case : 

- 1 < A <= 10  AND 2 <= A < 11,  优化为 2 <= A <= 10
- 1 < A  AND A < 0  ,  优化为 FalseCondition
- A > 1 OR A < 3，优化为 TrueCondition
- A > 1 OR A > 3，优化为 A > 1
- A > 1 or A = 5， 优化为 A > 1

**合并范围查询，可以减少规则计算的复杂度，尽可能优化为一个区间的范围查询。**

### 分片（Shard）计算

```yaml
    executors:
      - name: redirect
        mode: shd
        config:
          transaction_timeout: 60000
          db_groups:
            - name: world_0
              load_balance_algorithm: RandomWeight
              data_sources:
                - name: world_0
                  weight: r10w10
            - name: world_1
              load_balance_algorithm: RandomWeight
              data_sources:
                - name: world_1
                  weight: r10w10
          logic_tables:
            - db_name: world
              table_name: city
              allow_full_scan: true
              sharding_rule:
                column: id
                sharding_algorithm: NumberMod
              topology:
                "0": 0-4
                "1": 5-9
    data_source_cluster:
      - name: world_0
        capacity: 10
        max_capacity: 20
        idle_timeout: 60s
        dsn: root:123456@tcp(dbpack-mysql1:3306)/world?timeout=10s&readTimeout=10s&writeTimeout=10s&parseTime=true&loc=Local&charset=utf8mb4,utf8
        ping_interval: 20s
        ping_times_for_change_status: 3
      - name: world_1
        capacity: 10
        max_capacity: 20
        idle_timeout: 60s
        dsn: root:123456@tcp(dbpack-mysql2:3306)/world?timeout=60s&readTimeout=60s&writeTimeout=60s&parseTime=true&loc=Local&charset=utf8mb4,utf8
        ping_interval: 20s
        ping_times_for_change_status: 3
```

上面的配置描述了逻辑表 city 的拓扑：

```yaml
topology:
  "0": 0-4
  "1": 5-9
```

即 `city` 表分为 10 张分片表，`city_0` 至 `city_9`，这 10 张分片表分布于两个 DB：

```
world_0: city_0, city_1, city_2, city_3, city_4
world_1: city_5, city_6, city_7, city_8, city_9 
```

根据上述拓扑，计算分片时，只需计算出表分片，即可知请求落在哪个 DB 分片。DBPack 现支持两种分片算法，分别是 `NumberMod` 取模分片、`NumberRange` 范围分片。上面的配置描述了 `NumberMod` 算法如何配置，`NumberRange` 需要额外的参数描述分片健的分布：

```yaml
sharding_rule:
   column: id
   sharding_algorithm: NumberRange
   config:
     "0": "1-500"
     "1": "500-1000"
     "2": "1000-1500"
     "3": "1500-2000"
     "4": "2000-2500"
     "5": "2500-3000"
     "6": "3000-3500"
     "7": "3500-4000"
     "8": "4000-4500"
     "9": "4500-5000"
```

在描述分片健的范围时，可以使用 `K`、`M` 分别表示千、万，比如：0-1000K，1000K-2000K，2000M-3000M。

### 自动生成主键

DBPack 支持自动生成主键，目前支持两种主键生成算法，一种是 SnowFlake 雪花算法，一种是 Segment 号段算法。

SnowFlake 雪花算法配置如下：

```yaml
logic_tables:
- db_name: world
  table_name: city
  allow_full_scan: true
  sharding_rule:
    column: id
    sharding_algorithm: NumberMod
  sequence_generator:
    type: snowflake
    config:
      worker_id: 1001
```

雪花算法需指定 worker_id。

Segment 号段算法配置如下：

```yaml
logic_tables:
- db_name: world
  table_name: city
  allow_full_scan: true
  sharding_rule:
    column: id
    sharding_algorithm: NumberMod
  sequence_generator:
    type: Segment
    config:
      # 初始值，default 0
      from: 10000
      # 号段步长, default 1000
      step: 1000
      # 数据库连接配置
      dsn: dsn: root:123456@tcp(dbpack-mysql1:3306)/world
```

该算法主键从 10000 起，一次取 1000 个主键放入缓存中，供业务使用。

使用自动生成主键功能，在插入时，如未明确指定主键，则会改写 SQL 请求，自动加入主键。例如：

```sql
INSERT INTO city (`id`, `name`, `country_code`, `district`, `population`) VALUES (10001, '´s-Hertogenbosch', 'NLD', 'Noord-Brabant', 129170);
```

上面的请求已经指定主键，则不会生成主键。

```sql
INSERT INTO city (`name`, `country_code`, `district`, `population`) VALUES ('´s-Hertogenbosch', 'NLD', 'Noord-Brabant', 129170);
```

如果配置了自动生成主键，则请求会被改写为：

```sql
INSERT INTO city_5(name,country_code,district,population,id) VALUES ('´s-Hertogenbosch','NLD','Noord-Brabant',129170,1108222910313111555)
```

**例子中的表名根据主键以及分片算法，改写为 city_5。**

### 影子表

影子表配置：

```yaml
executors:
- name: redirect
  mode: shd
  config:
    transaction_timeout: 60000
    db_groups:
    - name: world_0
      load_balance_algorithm: RandomWeight
      data_sources:
        - name: world_0
          weight: r10w10
    - name: world_1
      load_balance_algorithm: RandomWeight
      data_sources:
        - name: world_1
          weight: r10w10
    global_tables:
    - country
    - countrylanguage
    - pt_city_0
    logic_tables:
    - db_name: world
      table_name: city
      allow_full_scan: true
      sharding_rule:
      column: id
      sharding_algorithm: NumberMod
      topology:
        "0": 0-4
        "1": 5-9
    # 影子表规则
    shadow_rules:
    - table_name: city
      # 计算影子表匹配规则的列
      column: country_code
      # 影子表匹配规则表达式，计算表达式的值时，%s 会替换为 column 的值。
      expr: "%s == \"US\""
      # 影子表前缀
      shadow_table_prefix: pt_
```

上面的配置表示逻辑表 `city` 启动了影子表路由功能，当 `country_code = "US"` 时，插入请求路由到影子表，影子表的前缀为 `pt_`。例如：

```sql
INSERT INTO city (`id`, `name`, `country_code`, `district`, `population`) VALUES (10, 'New York', 'US', 'Queens', 129170)
```

根据上面的 `NumberMod` 分片配置，该请求会被路由到 `city_0`，同时，满足 `country_code = "US"` 的条件，则请求最终被路由到 `pt_city_0` ：

```Sql
INSERT INTO pt_city_0 (`id`, `name`, `country_code`, `district`, `population`) VALUES (10, 'New York', 'US', 'Queens', 129170)
```

#### 影子表匹配表达式规则

1. 影子表匹配规则表达式对于字符类型，支持：

- `matches` (regex match)
- `contains` (string contains)
- `startsWith` (has prefix)
- `endsWith` (has suffix)

如：

```yaml
column: brand
expr: "%s matches \"h.*\""
```

当 branch 的值是 `hermes` 时，表达式的匹配结果为 `true`，请求将会路由到影子表。

再例如：

```yml
column: madein
expr: "%s contains \"china\""
```

当 madein 的值是 `china sichuan` 时，表达式的匹配结果为 `true`，请求将会路由到影子表。

2. 对于数字类型，支持一些基本的运算：

- `+` (addition)
- `-` (subtraction)
- `*` (multiplication)
- `/` (division)
- `%` (modulus)
- `^` or `**` (exponent)

如：

```yaml
column: userid
expr: "%s % 10 == 1"
```

当 userid 的值为 1689391 时，表达式的匹配结果为 `true`，请求将会路由到影子表。

3. 支持如下比较操作符：

- `==` (equal)
- `!=` (not equal)
- `<` (less than)
- `>` (greater than)
- `<=` (less than or equal to)
- `>=` (greater than or equal to)

影子表功能还支持通过在 SQL 中加入 `Shadow()` hint 的方式，显示声明请求是否路由到影子表，例如：

```sql
INSERT /*+ Shadow() */ INTO city (`id`, `name`, `country_code`, `district`, `population`) VALUES (20, '´s-Hertogenbosch', 'NLD', 'Noord-Brabant', 129170)
```

如果请求中包含 `Shadow()` hint，则不再进行影子表匹配规则的计算。


