# 基本使用
启动和关闭数据库server在各个操作系统上命令略有区别，用的时候查询即可；另外用户管理等操作不常用，也不记录笔记。

## 数据库操作
1. 连接服务器
```shell
# -h 和 -P 指定 MySQL 服务的 IP 地址和端口，本地可以不设置
# 回车之后输入密码
mysql -h ip地址 -P 端口 -u 用户名 -p
```

2. 其他操作
* 查看数据库：`SHOW DATABASES;`
* 创建数据库：`CREATE DATABASE 数据库名;`
* 删除数据库：`DROP DATABASE 数据库名;`
* 选择数据库：`USE 数据库名;`
* 查看数据库的状态信息：`SHOW STATUS;`
* 显示当前事件、用户、版本：`SELECT now();`、`SELECT user();`、`SELECT version();`
* ......
	

## 数据表操作
* 查看数据表：`SHOW TABLES;`
* 创建数据表：
```sql
CREATE TABLE table_name (
    column1 datatype,
    column2 datatype,
    ...
);
```
* 修改表结构，例如表名、字段名称、类型等操作：`ALTER ...`
* 更新数据：`UPDATE ...`
* 删除数据：`DELETE ...`
* 查询语句、`WHERE`、`LIKE`（模糊匹配） 使用时查找用法即可
* 排序：`ORDER BY`
* 分组：`GROUP BY`

### UNION
`UNION` 的作用是将多个查询结果的行堆叠起来，相当于纵向合并，例如下面的查询语句就会把`students`和`teachers`中的名称列数据合并起来
```sql
SELECT student_name AS name FROM students
UNION
SELECT teacher_name AS name FROM teachers;
```

> `UNION` 会去重，`UNION ALL` 不去重

### 连接
`JOIN`分为3种：
* `INNER JOIN`：默认行为，和`JOIN`一样，只返回两个表中完全匹配的行，也就是交集
* `LEFT JOIN`：以左边的表（FROM 后面的表）为主，右表能匹配上的就显示，匹配不上的填 NULL
* `RIGHT JOIN`：和左连接完全相反

举个例子，假设`users`表有张三、李四、王五，但`orders`表只有张三和李四有订单记录。

先使用内连接查询，
```sql
SELECT users.name, orders.product
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```


|name|product|
|---|---|
|张三|手机|
|张三|耳机|
|李四|电脑|


然后使用左连接查询：
```sql
SELECT users.name, orders.product
FROM users
LEFT JOIN orders ON users.id = orders.user_id;
```
|name|product|
|---|---|
|张三|手机|
|张三|耳机|
|李四|电脑|
|王五|NULL|

> 可以连接多个表，还可以配合 `WHERE` 筛选

### REGEXP 正则表达式
```sql
SELECT 字段名 FROM 表名 WHERE 字段名 REGEXP '正则规则';
```

## 事务
基本使用
```sql
-- 1. 开始事务 （START 也可以）
BEGIN TRANSACTION;

-- 2. 执行一些SQL语句
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;

-- 3. 判断是否要提交还是回滚
IF (条件) THEN
    COMMIT; -- 提交事务
ELSE
    ROLLBACK; -- 回滚事务
END IF;
```

### 保存点 (Savepoint)
如果在一个事务里执行了很多步骤，但只想回滚其中一部分，可以使用保存点。这就像玩游戏时的“存档”。

```sql
BEGIN TRANSACTION;

-- 步骤 1
UPDATE account SET money = money - 100 WHERE id = 1;

-- 设置一个存档点，叫 'step1_done'
SAVEPOINT step1_done; 

-- 步骤 2
INSERT INTO log (content) VALUES ('转账记录');

-- 假设步骤 2 出错了，或者你想撤销步骤 2，但不想影响步骤 1
ROLLBACK TO step1_done; 

-- 此时，步骤 1 的扣款依然有效，但步骤 2 的插入被撤销了。

COMMIT;
```

## 索引
索引的核心本质是空间换时间，创建和维护索引需要时间和空间成本，比如每次增删改都要更新索引。

比如一个表有10000行，那么每次查询都要全表遍历，时间复杂度为N，假如我们先用 HashMap 存储一下，时间复杂度可以变为 1。但哈希算法并不能支持范围查找，不适合作为数据库的索引数据结构。还有二叉查找树有顺序，可以满足范围查找，但极端情况会退化为链表，降低查找速度。还有一些数据结构不做介绍，总之，最终选择了B+树，它在查找和范围查找都有良好的性能。

B+树所有数据节点都是按照键值的大小存放在同一层的叶子节点上，而非叶子结点只存储key的信息。

### 创建索引
创建表时直接定义
```sql
-- 示例：用户表，包含主键、唯一索引、普通索引、联合索引
CREATE TABLE user (
    id INT PRIMARY KEY AUTO_INCREMENT,  -- 主键索引（自动创建）
    phone VARCHAR(11) UNIQUE,           -- 唯一索引
    name VARCHAR(20),
    age INT,
    gender CHAR(1),
    -- 普通索引
    INDEX idx_name (name),
    -- 复合索引
    INDEX idx_name_age (name, age)
);
```

已存在的表添加索引
```sql
-- 1. 普通索引
CREATE INDEX idx_age ON user(age);

-- 2. 唯一索引
CREATE UNIQUE INDEX idx_phone ON user(phone);

-- 3. 复合索引
CREATE INDEX idx_name_age_gender ON user(name, age, gender);

-- 4. 主键索引（仅无主键时添加）
ALTER TABLE user ADD PRIMARY KEY (id);
```

### 区别
* 主键索引：执行 PRIMARY KEY 约束时，MySQL 就会自动创建主键索引。
* 唯一索引：插入一条数据时，数据库会先检查这个值是否已经存在。如果存在，直接抛出 Duplicate entry 错误，阻止写入。（数据库为了实现“唯一约束”，在底层“偷偷”使用了“唯一索引”作为手段，所以唯一约束和唯一索引是关联的，创建一个唯一约束时，MySQL 会自动在后台创建一个唯一索引）
* 普通索引：没有约束的普通索引
* 复合索引：假设创建了一个复合索引`INDEX(a, b)`，索引会先按 a 排序，a 相同时才按 b 排序。所以查询时也会从最左边开始匹配，比如 `a = 1` 和 `a = 1 AND b = 2` 都会走索引，但 `b = 2` 不会走索引。


## 锁表
mysql 支持锁整个表，也可以锁行。

在日常开发中，我们主要使用“事务”，而“锁”是事务在背后自动为我们做的事情。开启一个事务，并执行 INSERT、UPDATE、DELETE 这些修改数据的操作时，MySQL（InnoDB引擎）会自动地、隐式地为你加上排他锁（X锁，也叫写锁）。

但例如遇到“先查询，再根据查询结果决定是否更新”的场景，并且对数据一致性要求极高时，仍然可能需要使用锁（不过也可以考虑在业务代码层加锁）。

## 触发器
触发器是一段存储在数据库中的 SQL 代码，不需要手动调用。当对某个表执行了特定的操作（如插入、更新、删除）时，数据库会自动执行这段代码。