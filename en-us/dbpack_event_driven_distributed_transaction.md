# EAT Mode

## 1. UndoLog

When DBPack is executing SQL, it will store the UndoLog of the SQL operation. The UndoLog will be removed asynchronously in case global transaction commit. In case global transaction rollback, the UndoLog will be executed asynchronously to compensate for the branch transaction.  

We can take following `departments` table for example, to illustrate the logic of generating UndoLog.   

```sql
CREATE TABLE `departments` (
  `id` bigint NOT NULL AUTO_INCREMENT,
  `dept_no` char(4) COLLATE utf8mb4_unicode_ci NOT NULL,
  `dept_name` varchar(100) COLLATE utf8mb4_unicode_ci NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `dept_name` (`dept_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

Suppose the application wants to execute bellow business SQL:

```sql
update departments set dept_name = 'moonlight' where dept_name = 'sunset';
```

DBPack will parse the SQL and get the `before image` of it, before above SQL been executed.

```sql
select id, dept_no, dept_name from departments where dept_name = 'sunset';
```

The obtained `before image` is like following:

| id   | dept_no | dept_name |
| ---- | ------- | --------- |
| 230  | 1001    | sunset    |

After the business SQL been executed, the `department` name will be updated to `moonlight`, then we can get the `after image` via the primary key of `before image`.

```sql
select id, dept_no, dept_name from departments where id = 230;
```

The obtained `after image` is like following:

| id   | dept_no | dept_name |
| ---- | ------- | --------- |
| 230  | 1001    | moonlight |

After `before image` and `after image` been serialized, we can get following data structure of UndoLog: 

```json
{
	"is_binary": true,
	"sql_type": "UPDATE",
	"schema_name": "employees",
	"table_name": "departments",
	"lock_key": "departments:230",
	"after_image": {
		"rows": [{
			"fields": [{
				"key_type": "pk",
				"name": "id",
				"type": 4,
				"value": 230
			}, {
				"name": "dept_no",
				"type": 12,
				"value": "1001"
			}, {
				"name": "dept_name",
				"type": 12,
				"value": "moonlight"
			}]
		}],
		"table_name": "departments"
	},
	"before_image": {
		"rows": [{
			"fields": [{
				"key_type": "pk",
				"name": "id",
				"type": 4,
				"value": 230
			}, {
				"name": "dept_no",
				"type": 12,
				"value": "1001"
			}, {
				"name": "dept_name",
				"type": 12,
				"value": "sunset"
			}]
		}],
		"table_name": "departments"
	}
}
```

We use Json data structure to display for convenient purpose. In DBpack, the UndoLog is serialized by Proto Buffer.


## 2. SQL Compensation

+ INSERT Operation

  Create DELETE compensation SQL through UndoLog

  ```sql
  DELETE FROM {table_name} WHERE id = ?
  ```

+ DELETE operation

  Create INSERT compensation SQL through UndoLog

  ```sql
  INSERT INTO {table_name} ({columns}) VALUES ({values})
  ```

+ UPDATE operation

  Create UPDATE compensation SQL through UndoLog

  ```
  UPDATE {table_name} SET {columns} = {values} WHERE id = ?
  ```



## 3. Process Flow

<img src="https://cectc.github.io/dbpack-doc/images/distributed-transaction-en.gif" alt="image-20220427100734991" style="zoom:67%;" />

+ After the request been sent to the aggregation service, the ETCD will write global transaction data, and the unique mark xid is generated. （example: gs/aggregationSvc/2612341069705662465）。

***

+ When calling order service, the `XID` is passed to order service context by the HTTP Header.

+ By injecting `XID` Hint into the SQL to be executed, DBPack will intercept the SQL and do proxy for it, for example:

  ```sql
  INSERT /*+ XID('%s') */ INTO order.so_master (sysno, so_id, buyer_user_sysno, seller_company_code, receive_division_sysno, receive_address, receive_zip, receive_contact, receive_contact_phone, stock_sysno, payment_type, so_amt, status, order_date, appid, memo) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,now(),?,?)
  ```

+ Execute business SQL.

+ DBPack sidecar intercepts the business SQL and check if it has the Hint tag. If yes, the DBPack will create prepare the `before image` and `after image` for that SQL, by inserting `UndoLog` into database with the local transaction. The ETCD will write `BranchTransaction` data, which is linked to the xid (example: gs/aggregationSvc/2612341069705662465/${branchID}), which means the xid has registered a branch transaction.

***

+ When calling product service, the `XID` is passed to product service context by the HTTP Header.

+ By injecting `XID` Hint into the SQL to be executed, DBPack will intercept the SQL and do proxy for it, for example:

  ```sql
  update /*+ XID('%s') */ product.inventory set available_qty = available_qty - ?, allocated_qty = allocated_qty + ? where product_sysno = ? and available_qty >= ?;
  ```

+ Execute business SQL.

+ DBPack sidecar intercepts the business SQL and check if it has the Hint tag. If yes, the DBPack will create prepare the `before image` and `after image` for that SQL, by inserting `UndoLog` into database with the local transaction. The ETCD will write `BranchTransaction` data, which is linked to the xid (example: gs/aggregationSvc/2612341069705662465/${branchID}), which means the xid has registered a branch transaction.

***

+ + Back to the aggregation service, the business logic has been executed. If there is no exception nor error, then DBPack will get `BranchTransaction` by `XID`, and update their branch status to `committing`. By ETCD watch, the Sidecar will know that the branch transaction can be committed, so the DBPack will delete `UndoLog` from the `undo_log` table. If there is error in business logic, then the DBPack will get `BranchTransaction` by `XID`, and update their branch status to `rolling back`. By ETCD watch, the Sidecar will know that the branch transaction should be rolled back, so the DBPack will execute `UndoLog` from the `undo_log` table to recover data to before transaction status.

<br>

From a global perspective, above process be summarised into following abstract steps:

```sql
# session1:
    START TRANSACTION;
        INSERT /*+ XID('gs/aggregationSvc/2612341069705662465') */ INTO order.so_master (sysno, so_id, buyer_user_sysno, seller_company_code, receive_division_sysno, receive_address, receive_zip, receive_contact, receive_contact_phone, stock_sysno, payment_type, so_amt, status, order_date, appid, memo) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,now(),?,?);
        INSERT /*+ XID('gs/aggregationSvc/2612341069705662465') */ INTO order.so_item(sysno, so_sysno, product_sysno, product_name, cost_price, original_price, deal_price, quantity) VALUES (?,?,?,?,?,?,?,?);
    COMMIT;

# session2:
    START TRANSACTION;
        UPDATE /*+ XID('gs/aggregationSvc/2612341069705662465') */ product.inventory set available_qty = available_qty - ?, allocated_qty = allocated_qty + ? WHERE product_sysno = ? and available_qty >= ? ;
    COMMIT;
```

The entire life cycle of the transaction is strung together through `XID`.
