# AT 模式

<img src="./images/image-20220427100734991.png" alt="image-20220427100734991" style="zoom:67%;" />

+ 请求发到聚合层服务后，在 etcd 写入全局事务数据，产生唯一标识 xid（比如：gs/aggregationSvc/2612341069705662465）。

***

+ 调用订单服务时，通过 HTTP Header 传递 `XID` 到订单服务上下文 (Context)。

+ 将 `XID` 用 `Hint` 的的方式，添加到要执行的 SQL 语句中。例如：

  ```sql
  INSERT /*+ XID('%s') */ INTO order.so_master (sysno, so_id, buyer_user_sysno, seller_company_code, receive_division_sysno, receive_address, receive_zip, receive_contact, receive_contact_phone, stock_sysno, payment_type, so_amt, status, order_date, appid, memo) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,now(),?,?)
  ```

+ 执行业务 SQL。

+ DBPack sidecar 拦截到要执行的业务 SQL，检测 SQL 上是否是否带有 Hint 标记。如果有，则将该 SQL 执行前后的数据构造成 `UndoLog` 记录下来，在 etcd 写入 `BranchTransaction` 数据，并跟 xid 关联一个 key (比如：gs/aggregationSvc/2612341069705662465/${branchid})，表示 xid 下存在一个事务分支。

***

+ 调用商品服务时，通过 HTTP Header 传递 `XID` 到商品服务上下文 (Context)。

+ 将 `XID` 用 `Hint` 的的方式，添加到要执行的 SQL 语句中。例如：

  ```sql
  update /*+ XID('%s') */ product.inventory set available_qty = available_qty - ?, allocated_qty = allocated_qty + ? where product_sysno = ? and available_qty >= ?
  ```

+ 执行业务 SQL。

+ DBPack sidecar 拦截到要执行的业务 SQL，检测 SQL 上是否是否带有 Hint 标记。如果有，则将该 SQL 执行前后的数据构造成 `UndoLog` 记录下来，在 etcd 写入 `BranchTransaction` 数据，并跟 xid 关联一个 key (比如：gs/aggregationSvc/2612341069705662465/${branchid2})，表示 xid 下存在一个事务分支。

***

+ 回到聚合层服务，业务逻辑执行完毕，如果业务执行成功没有异常，则根据 `XID` 获取 `BranchTransaction` 修改分支状态为提交，sidecar watch 到分支事务 key 变化立即删除 `UndoLog`；如果业务执行失败，则根据 `XID` 获取 `BranchTransaction` 修改分支状态为回滚，sidecar watch 到分支事务 key 变化根据 `UndoLog` 生成反向回滚语句。

<br>

站在全局的视角，上述的过程，可抽象为下面的步骤：

```sql
session1:
    START TRANSACTION
        INSERT /*+ XID('gs/aggregationSvc/2612341069705662465') */ INTO order.so_master (sysno, so_id, buyer_user_sysno, seller_company_code, receive_division_sysno, receive_address, receive_zip, receive_contact, receive_contact_phone, stock_sysno, payment_type, so_amt, status, order_date, appid, memo) VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,now(),?,?)
        INSERT /*+ XID('gs/aggregationSvc/2612341069705662465') */ INTO seata_order.so_item(sysno, so_sysno, product_sysno, product_name, cost_price, original_price, deal_price, quantity) VALUES (?,?,?,?,?,?,?,?)
    COMMIT

session2:
    START TRANSACTION
        UPDATE /*+ XID('gs/aggregationSvc/2612341069705662465') */ product.inventory set available_qty = available_qty - ?, allocated_qty = allocated_qty + ? WHERE product_sysno = ? and available_qty >= ?
    COMMIT
```

通过 `XID` 串起了事务的整个生命周期。

