### 1.
- mysql选用repeatable read或者read committed隔离级别
- 一个例子：
我们有一个表

|id | product | quantity|
|:-|:-|:-|
|1  | apple   | 100|

- 事务一
```mysql 
START TRANSACTION;
SELECT quantity FROM orders WHERE product = 'apple'; -- 返回100
UPDATE orders SET quantity = 50 WHERE product = 'apple';
COMMIT;

```
- 事务二
```MYSQL
START TRANSACTION;
SELECT quantity FROM orders WHERE product = 'apple'; -- 在T1提交前，返回100
-- 在T1提交后
SELECT quantity FROM orders WHERE product = 'apple'; -- 返回50
COMMIT;

```

- 更改事务隔离级别
```mysql
-- 事务隔离级别为RR
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 事务隔离级别为RC
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;


```

- 查看事务隔离级别
```MYSQL
-- 查看当前会话的隔离级别
SELECT @@transaction_isolation;
-- 查看系统当前的隔离级别
SELECT @@global.transaction_isolation;

```

### 2.
- 我们先创建集合并插入数据
```javascript
db.createCollection("orders");
db.orders.insertOne({
  product: "apple",
  quantity: 100
})

```
- 事务一
```JavaScript
var session = db.getMongo().startSession();
session.startTransaction({readConcern: { level: 'majority' },writeConcern: { w: 'majority' }})
db.orders.find()
db.orders.updateOne(
  { "_id" : ObjectId("65759dce4fa2039de1b6910f") }, 
  { $set: { "quantity" : 50 } }
)

session.endSession()
```
- 事务二
```javascript
var session = db.getMongo().startSession();
session.startTransaction({readConcern: { level: 'majority' },writeConcern: { w: 'majority' }});
db.orders.find()
db.orders.find()
session.endSession()
```