# Sharding Databases && Tables

> Currently, DBPack supports following functions: cross-shard query, cross-DB query, Order By, Limit, automatic primary key generation, shadow table and so on. Here is the details:

## Optimizer

The SQL request passed to DBPack, will be transformed into AST abstract syntax tree by SQL Parser. Then, the Optimizer will make a execution plan for this SQL. Currently, the Optimizer supports following logic:

### WHERE Condition Pre-Process

1.  function pre-calculation

- SELECT 1 + 1 ,  will be optimized to: SELECT 2

- SELECT 1 = 1 ,  will be optimized to: SELECT 1

- SELECT 0 = 1 ,  will be optimized to: SELECT 0

- string functions:

  ```LENGTH、CHAR_LENGTH、CHARACTER_LENGTH、CONCAT、CONCAT_WS、LEFT、LOWER、LPAD、LTRIM、REPEAT、REPLACE、REVERSE、RIGHT、RPAD、RTRIM、SPACE、STRCMP、SUBSTRING、UPPER```

- mathematics functions:

  ```ABS、CELL、EXP、FLOOR、MOD、PI、POW、POWER、RAND、ROUND、SIGN、SQRT```

- datetime functions:

  ```ADDDATE、CURDATE、CURRENT_DATE、CURRENT_TIMESTAMP、CURTIME、DATEDIFF、DATE_FORMAT、DAY、DAYNAME、DAYOFMONTH、DAYOFWEEK、DAYOFYEAR、FROM_UNIXTIME、HOUR、LOCALTIME、MINUTE、MONTHNAME、MONTH、NOW、QUARTER、SECOND、SUBDATE、SYSDATE、UNIX_TIMESTAMP```

- CAST functions:

  ```CAST、CONVERT```

- other functions:

  ```IF、IFNULL、MD5、SHA、SHA1```

DBPack can use the return values of these functions for calculating sharding, for example:

```
SELECT film_id, title, rental_rate, rental_duration FROM film WHERE rental_duration = POW(2,2)
```

Provided using 4 as sharding pattern of `rental_duration`, we can know that the sharding result will be 0, hence the query can be refactored to following SQL:

```
SELECT film_id, title, rental_rate, rental_duration FROM film_0 WHERE rental_duration = POW(2,2)
```

We can get correct query result by executing above SQL.

2. optimize ture / false checking

For example:

- false or a = 1, will be optimized to: a = 1
- true or a = 1，will be optimized to: TrueCondition
- false and a =1 , will be optimized to: FalseCondition
- true and a = 1 , will be optimized to: a = 1
- 1  and a = 1， will be optimized to: a = 1  (non-zero constant expression represents true, 0 represents for false)

**TrueCondition indicates full table scanning; FalseCondition indicates that there is no sharding result for this SQL, in other words, empty result**

3. smartly merging and/or query

some cases:

- 1 < A <= 10  AND 2 <= A < 11,  will be optimized to: 2 <= A <= 10
- 1 < A  AND A < 0  ,  will be optimized to: FalseCondition
- A > 1 OR A < 3，will be optimized to: TrueCondition
- A > 1 OR A > 3，will be optimized to: A > 1
- A > 1 or A = 5， will be optimized to: A > 1

**By merging and/or query, it can reduce the complexity of rule calculation, and optimize queries into certain scope as much as possible. **

### Sharding calculation

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

Above configuration describes following topology of `city` table.

```yaml
topology:
  "0": 0-4
  "1": 5-9
```

In other words, `city` table will be split into 10 sharding tables, from `city_0` to `city_9`. And these tables are be distributed on 2 DB:

```
world_0: city_0, city_1, city_2, city_3, city_4
world_1: city_5, city_6, city_7, city_8, city_9 
```

According to the above topology, when calculating sharding, DBPack only need to calculate the table sharding to know on which DB shard the request falls. DBPack now supports two sharding algorithms, namely `NumberMod` fragmentation sharing and `NumberRange` range sharding. The above configuration describes how to configure the `NumberMod` algorithm, and `NumberRange` requires additional parameters to describe the distribution of the partition keys:

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

When describing the scope of sharding keys, we can use `K`, `M`, representing for kilo and 10 kilo, such as 0-1000K，1000K-2000K，2000M-3000M.

### Automatically generate primary key

DBPack can automatically generate primary key. Currently, it supports 2 algorithms: SnowFlake and Segment number.

Following is the SnowFlake algorithm configuration example:

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

SnowFlake algorithm requires specifying worker_id 

Here is the configuration example of Segment number algorithm: 

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
      # default value, default is 0
      from: 10000
      # step value, default is 1000
      step: 1000
      # database connection configuration
      dsn: root:123456@tcp(dbpack-mysql1:3306)/world
```

The primary key of this algorithm starts from 10000, and 1000 primary keys are taken into the cache at a time for business logic using.

When using automatic primary key generation, if the primary key is not explicitly specified when inserting, the SQL request will automatically add the primary key. E.g:

```sql
INSERT INTO city (`id`, `name`, `country_code`, `district`, `population`) VALUES (10001, '´s-Hertogenbosch', 'NLD', 'Noord-Brabant', 129170);
```

If the above request has specified a primary key, the primary key will not be generated.

```sql
INSERT INTO city (`name`, `country_code`, `district`, `population`) VALUES ('´s-Hertogenbosch', 'NLD', 'Noord-Brabant', 129170);
```

If automatic primary key generation is configured, the request will be overwritten into:

```sql
INSERT INTO city_5(name,country_code,district,population,id) VALUES ('´s-Hertogenbosch','NLD','Noord-Brabant',129170,1108222910313111555)
```

**The table name in the example is rewritten as city_5 according to the primary key and fragmentation algorithm.**

### shadow table

shadow table configuration:

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
    # shadow table rules
    shadow_rules:
    - table_name: city
      # calculate the columns of shadow table matching rules
      column: country_code
      # The regular expression of shadow table. When calculating the value of the expression, %s will be replaced with the column value.
      expr: "%s == \"US\""
      # shadow table prefix
      shadow_table_prefix: pt_
```

The above configuration indicates that the logical table `city` has enabled the shadow table routing function. When `country_code = "US"`, the insert request is routed to the shadow table, and the prefix of the shadow table is `pt_`. E.g:

```sql
INSERT INTO city (`id`, `name`, `country_code`, `district`, `population`) VALUES (10, 'New York', 'US', 'Queens', 129170)
```

According to the `NumberMod` sharding configuration above, the request will be routed to `city_0`. At the same time, if the condition of `country_code = "US"` is met, the request will be routed to `pt_city_0`:

```Sql
INSERT INTO pt_city_0 (`id`, `name`, `country_code`, `district`, `population`) VALUES (10, 'New York', 'US', 'Queens', 129170)
```

#### expression rules of shadow table

1. string match:

- `matches` (regex match)
- `contains` (string contains)
- `startsWith` (has prefix)
- `endsWith` (has suffix)

for example:

```yaml
column: brand
expr: "%s matches \"h.*\""
```

When the value of branch is `hermes`, the matching result of the expression is `true`, and the request will be routed to the shadow table.

another example:

```yml
column: madein
expr: "%s contains \"china\""
```

When the value of madein is `china sichuan`, the matching result of the expression is `true`, and the request will be routed to the shadow table.

2. for numeric types, it supports some basic operations:

- `+` (addition)
- `-` (subtraction)
- `*` (multiplication)
- `/` (division)
- `%` (modulus)
- `^` or `**` (exponent)

for example:

```yaml
column: userid
expr: "%s % 10 == 1"
```

When userid value is 1689391, the matching result of the expression is `true`, and the request will be routed to the shadow table.

3. comparison operators:

- `==` (equal)
- `!=` (not equal)
- `<` (less than)
- `>` (greater than)
- `<=` (less than or equal to)
- `>=` (greater than or equal to)

The shadow table function also supports adding `Shadow()` hint in SQL to display whether the statement request is routed to the shadow table, for example:

```sql
INSERT /*+ Shadow() */ INTO city (`id`, `name`, `country_code`, `district`, `population`) VALUES (20, '´s-Hertogenbosch', 'NLD', 'Noord-Brabant', 129170)
```

If the request has a `Shadow()` hint, no shadow table matching rules will be calculated.
