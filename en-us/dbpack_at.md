# AT Mode

<img src="https://cectc.github.io/dbpack-doc/images/distributed-transaction-en.gif" alt="image-20220427100734991" style="zoom:67%;" />

+ After the request been sent to the aggregation service, the ETCD will write global transaction data, and the unique mark xid is generated. （example: gs/aggregationSvc/2612341069705662465）。

***

+ When calling order service, the `XID` is passed to order service context by the HTTP Header.

+ By injecting `XID` Hint into the SQL to be executed, DBPack will intercept the SQL and do proxy for it, for example:

  ```sql
  INSERT /*+ XID('%s') */ INTO order.so_master (sysno, so_id, buyer_user_sysno, seller_company_code, receive_division_sysno, receive_address, receive_zip, receive_contact, receive_contact_phone, stock_sysno, payment_type, so_amt, status, order_date, appid, memo) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,now(),?,?)
  ```

+ Execute business SQL.

+ DBPack sidecar intercepts the business SQL and check if it has the Hint tag. If yes, the DBPack will create prepare the `before image` and `after image` for that SQL, by inserting `UndoLog` into database with the local transaction. The ETCD will write `BranchTransaction` data, which is linked to the xid (example: gs/aggregationSvc/2612341069705662465/${branchid}), which means the xid has registered a branch transction.

***

+ When calling product service, the `XID` is passed to product service context by the HTTP Header.

+ By injecting `XID` Hint into the SQL to be executed, DBPack will intercept the SQL and do proxy for it, for example:

  ```sql
  update /*+ XID('%s') */ product.inventory set available_qty = available_qty - ?, allocated_qty = allocated_qty + ? where product_sysno = ? and available_qty >= ?
  ```

+ Execute business SQL.

+ DBPack sidecar intercepts the business SQL and check if it has the Hint tag. If yes, the DBPack will create prepare the `before image` and `after image` for that SQL, by inserting `UndoLog` into database with the local transaction. The ETCD will write `BranchTransaction` data, which is linked to the xid (example: gs/aggregationSvc/2612341069705662465/${branchid}), which means the xid has registered a branch transction.

***

+ + Back to the aggregation service, the business logic has been executed. If there is no exception nor error, then DBPack will get `BranchTransaction` by `XID`, and update their branch status to `committing`. By ETCD watch, the Sidecar will know that the branch transaction can be commited, so the DBPack will delete `UndoLog` from the `undo_log` table. If there is error in business logic, then the DBPack will get `BranchTransaction` by `XID`, and update their branch status to `rolling back`. By ETCD watch, the Sidecar will know that the branch transaction should be rolled back, so the DBPack will execute `UndoLog` from the `undo_log` table to recover data to before transaction status.

<br>

From a global perspective, above process be can abstracted into following steps:

```sql
session1:
    START TRANSACTION
        INSERT /*+ XID('gs/aggregationSvc/2612341069705662465') */ INTO order.so_master (sysno, so_id, buyer_user_sysno, seller_company_code, receive_division_sysno, receive_address, receive_zip, receive_contact, receive_contact_phone, stock_sysno, payment_type, so_amt, status, order_date, appid, memo) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,now(),?,?)
        INSERT /*+ XID('gs/aggregationSvc/2612341069705662465') */ INTO order.so_item(sysno, so_sysno, product_sysno, product_name, cost_price, original_price, deal_price, quantity) VALUES (?,?,?,?,?,?,?,?)
    COMMIT

session2:
    START TRANSACTION
        UPDATE /*+ XID('gs/aggregationSvc/2612341069705662465') */ product.inventory set available_qty = available_qty - ?, allocated_qty = allocated_qty + ? WHERE product_sysno = ? and available_qty >= ?
    COMMIT
```

The entire life cycle of the transaction is strung together through `XID`.
