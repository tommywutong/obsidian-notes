---
title: 【iOS】SQLite 事务与索引、FMDB 的并发模型、Core Data 的代价
published: 2026-07-27
description: "「插入要开事务」人人都知道，但快多少取决于 journal 模式：rollback journal 下是 870 倍，WAL 下只剩 43 倍。索引也一样，同一条 WHERE，命中 1 行快 626 倍，命中 1 万行只快 1.1 倍。"
tags:
  - iOS
  - SQLite
  - FMDB
  - CoreData
  - 持久化
category: iOS
series: 2026 暑假 iOS 底层学习
seriesSlug: ios-internals-2026-summer
seriesOrder: 35
draft: true
---
# SQLite 事务与索引、FMDB 的并发模型、Core Data 的代价

先看一组数字。同一台机器、同一个表、同样插 10000 条记录：

```text
journal_mode=delete
不开事务（每条一个隐式事务）  3589.6 ms
一个大事务                        4.0 ms      901x
```

```text
journal_mode=wal
不开事务（每条一个隐式事务）   162.4 ms
一个大事务                        3.8 ms       43x
```

两组的"开事务"耗时几乎一样，都是 4 毫秒。差别全在"不开事务"那一行：一个 3.6 秒，一个 0.16 秒。

**「插入要开事务」这句话的收益是 40 倍还是 900 倍，取决于你有没有开 WAL。** 而绝大多数讲事务的文章不提 journal 模式。这一篇里所有数字都是我在 macOS arm64 上现跑的，程序和跑法在文末，你可以自己复现。

存储介质怎么选（沙盒目录、plist、NSUserDefaults、Keychain）在 [[iOS 持久化选型：沙盒、plist、NSUserDefaults 与 Keychain]]，对象怎么编解码在 [[iOS 序列化：NSCoding、NSSecureCoding 与 JSON 三条路]]。这一篇只走数据库这一条路。

---

## 一、那 900 倍花在哪

SQLite 的文档把隐式事务说得很明白：

> No reads or writes occur except within a transaction. Any command that accesses the database (basically, any SQL command, except a few PRAGMA statements) will automatically start a transaction if one is not already in effect. Automatically started transactions are committed when the last SQL statement finishes.

不开事务不等于没有事务，等于开了 10000 个事务。每个都要走完整的提交流程。

我一开始以为慢的原因就是 fsync，把 `synchronous` 关掉验证了一下：

```text
10000 条裸插入（全程不开显式事务），5 轮取中位
delete + synchronous=FULL      6113.7 ms
delete + synchronous=OFF       1955.3 ms
wal    + synchronous=FULL       663.0 ms
wal    + synchronous=NORMAL     168.8 ms
wal    + synchronous=OFF        116.0 ms
```

关掉 fsync 之后 delete 模式还剩将近 2 秒。所以 fsync 只吃掉了三分之二。另外三分之一在别处。它是 rollback journal 本身的开销：每个事务都要创建 `-journal` 文件、写入原始页、提交后再删掉它。10000 次文件创建加删除，这笔账在任何文件系统上都不便宜。

WAL 模式没有这一步。它只往 `-wal` 文件尾部追加，事务边界靠 commit 记录标记，文件本身一直在那儿。这就是同样是"不开事务"，WAL 快 20 倍的原因。

事务粒度的完整表：

| 粒度 | delete 模式 | WAL 模式 |
|---|---|---|
| 不开事务 | 3589.6 ms | 162.4 ms |
| 一个大事务 | 4.0 ms | 3.8 ms |
| 每 1000 条一个事务 | 7.3 ms | 4.1 ms |
| 每 100 条一个事务 | 39.0 ms | 5.5 ms |

从 1000 条一批降到 100 条一批，delete 模式劣化 5 倍，WAL 只劣化 1.3 倍。批大小这个参数在 WAL 下基本不用调。

关于那 901 倍，得补一句方法论上的话。我一共跑了五轮，第一轮测出来是 1540 倍，后面四轮稳定在 760~900。第一轮是冷启动，页缓存和 SSD 的写入状态都不一样。这个系列吃过只跑两次就下结论的亏，所以我把区间写出来：**delete 模式下事务的收益是 800 到 900 倍这个量级，不是一个精确数字。**

---

## 二、索引：收益由选择性决定

10 万行的 `person(id, name, city, age)`，`name` 唯一，`city` 只有 10 个不同值，`age` 有 60 个。三个列各建一个索引，跑 `ANALYZE`。

```text
                                  无索引        有索引       加速
name='person050000'   (1 行)     4.3089 ms   0.0069 ms    626.7x
age=30              (1667 行)    4.1500 ms   0.7375 ms      5.6x
city='Hangzhou'     (1 万行)     5.4650 ms   4.8660 ms      1.1x
```

同样是等值查询，同样走了索引，加速从 626 倍掉到 1.1 倍。`EXPLAIN QUERY PLAN` 三条都是 `SEARCH`：

```text
sqlite> EXPLAIN QUERY PLAN SELECT id,city FROM person WHERE name='person050000';
QUERY PLAN
`--SEARCH person USING INDEX idx_name (name=?)
sqlite> EXPLAIN QUERY PLAN SELECT id,name FROM person WHERE city='Hangzhou';
QUERY PLAN
`--SEARCH person USING INDEX idx_city (city=?)
```

执行计划看着一模一样，性能差 500 倍。原因在回表。索引查完只拿到 rowid，还要回表取 `name` 列。命中 1 万行就是 1 万次回表，每次都是一趟 B-tree 随机查找，加起来比顺序扫一遍表还贵一点点。SQLite 的查询规划器在这里其实选错了，即使跑过 `ANALYZE` 也还是选了索引。

那 `city` 上这个索引就完全没用吗？改一个字就有用：

```text
SELECT id,name FROM person WHERE city='Hangzhou'   4.8660 ms    1.1x
SELECT id      FROM person WHERE city='Hangzhou'   0.4975 ms   10.2x
SELECT count(*) FROM person WHERE city='Hangzhou'  0.2248 ms   20.3x
```

```text
`--SEARCH person USING COVERING INDEX idx_city (city=?)
```

`COVERING INDEX`。只取 `id` 的时候不用回表。因为 `id` 是 `INTEGER PRIMARY KEY`，也就是 rowid，索引项里本来就带着。回表这一步一消失，10 倍就出来了。

`ORDER BY name LIMIT 10` 是另一个方向：无索引 6.09 ms，有索引 0.0046 ms，1338 倍。排序类查询的索引收益比等值查询大得多，因为索引本身就是有序的，取前 10 条读完就停。

### 索引的代价

10 万行、单个大事务、WAL：

```text
无索引              插入  45.8 ms
先建索引再插入      插入 112.3 ms   （慢 2.45 倍）
先插入再建索引      插入 166.8 ms + 建索引 59.5 ms = 226.3 ms
```

```text
无索引  3203072 bytes
有索引  4984832 bytes   （涨 55.6%）
```

一个索引，插入慢 2.4 倍，文件大 56%。这个比例比我预期的高，因为 `city` 是变长文本。

顺便说，"先插入再建索引更快"这条经验在这里没成立。第三行加起来 226 ms，比先建索引的 112 ms 还慢一倍。我猜是因为建索引要重新排一遍 10 万行并额外写一遍文件，而边插边建时数据已经在页缓存里。这个结论只在这个规模上验过，别当通用规律用。

### 建了索引却用不上的四种写法

```text
                                   无索引     有索引    加速
LIKE '%050000'（前置通配）        6.54 ms   7.13 ms   0.9x
LIKE 'person0500%'（后置通配）    6.58 ms   5.66 ms   1.2x
lower(name)='person050000'        9.63 ms   8.23 ms   1.2x
age+0=30                          3.84 ms   2.89 ms   1.3x
```

四条的执行计划都退化成扫描：

```text
sqlite> EXPLAIN QUERY PLAN SELECT id FROM person WHERE lower(name)='person050000';
QUERY PLAN
`--SCAN person USING COVERING INDEX idx_name
```

`SCAN person USING COVERING INDEX` 这个措辞很容易骗人。它有 `INDEX` 三个字，但动词是 `SCAN` 不是 `SEARCH`。意思是"整个索引从头扫到尾"，只不过因为索引比表窄，扫索引比扫表稍微省一点 I/O。看执行计划要看动词。

前三条里，第一条和第三条是常识：前置通配和对列做函数运算，索引都用不上。第二条不是。`LIKE 'person0500%'` 是标准的前缀匹配，理论上应该能转成范围查询。它没转。

原因是 SQLite 的 `LIKE` 默认对 ASCII 大小写不敏感：

```text
sqlite> SELECT 'ABC' LIKE 'abc';
1
```

而 `idx_name` 是按二进制序排的。大小写不敏感的匹配没法映射成二进制序上的一个区间，优化就不能做。打开 `case_sensitive_like`：

```text
后置通配 + PRAGMA case_sensitive_like=ON   5.73 ms → 0.0086 ms   670x
`--SEARCH person USING COVERING INDEX idx_name (name>? AND name<?)
```

670 倍。区间条件 `name>? AND name<?` 就是 LIKE 优化转出来的。

改全局 pragma 会影响所有查询，风险太大。实践中的做法是给列单独建一个大小写不敏感的索引：

```sql
CREATE INDEX idx_name_nc ON person(name COLLATE NOCASE);
```

```text
sqlite> EXPLAIN QUERY PLAN SELECT id FROM person WHERE name LIKE 'person0500%';
QUERY PLAN
`--SEARCH person USING COVERING INDEX idx_name_nc (name>? AND name<?)
```

这条我第一次跑的时候还踩了个坑。我原本把测试数据命名成 `person_050000`，跑出来加了 `case_sensitive_like` 还是很慢。想了半天才反应过来：`_` 在 LIKE 里是单字符通配符，`'person_0500%'` 的可用前缀只有 `person`，范围条件覆盖全表。这是典型的仪器问题——改了变量名，结果就对了。

### 一条和 MySQL 经验冲突的

MySQL 圈子里流传很广的一条是"给整数列传字符串会导致索引失效"。SQLite 上不是这样：

```text
age='30'（字符串字面量）   无索引 4.73 ms   有索引 0.69 ms   6.8x
`--SEARCH person USING INDEX idx_age (age=?)
```

索引正常使用，结果也对（1667 行）。因为 `age` 列有 INTEGER 亲和性，SQLite 在比较前把 `'30'` 转成整数 30，转换发生在值上不是列上。InnoDB 的规则不能直接套过来。

---

## 三、WAL：并发行为到底改了什么

先把常见的说法摆出来：rollback journal 下写会阻塞读，WAL 下不会。

我按这句话设计了第一个实验：A 连接 `BEGIN IMMEDIATE` 写一行不提交，B 连接去读。

```text
=== journal_mode=delete ===
  A 已 BEGIN IMMEDIATE 并写入一行，尚未 COMMIT
  B 读:  rc=100  count=3        ← 读成功了
  C 写:  rc=5    database is locked
```

B 读成功了。rollback journal 模式下，写事务进行中只持有 RESERVED 锁，读者照样能读，读到的是修改前的内容。要等到 COMMIT 那一刻才升级到 EXCLUSIVE，那个窗口很短。所以"写阻塞读"这个说法在这种最简单的场景下并不成立。

反过来做才有区别。让 R 连接开一个读事务、游标停在中间，然后 W 连接去写：

```text
delete   W 开写事务并 INSERT: rc=0   ok
delete   W COMMIT:            rc=5   database is locked
delete   R 这轮游标一共读到 3 行

wal      W 开写事务并 INSERT: rc=0   ok
wal      W COMMIT:            rc=0   ok
wal      R 这轮游标一共读到 3 行
```

delete 模式下 W 的 COMMIT 被卡住了，因为 R 的 SHARED 锁挡着 EXCLUSIVE 升级。WAL 模式下 COMMIT 直接成功，而 R 继续读到的还是 3 行——它拿着自己那个快照，新写入的行对它不可见。

官方文档的措辞是：

> WAL provides more concurrency as readers do not block writers and a writer does not block readers. Reading and writing can proceed concurrently.

关键在于"a writer"是单数。WAL 允许一写多读，不允许多写。两个写事务并发，后来的照样 `SQLITE_BUSY`。这一点和 rollback journal 没区别。

### -wal 和 -shm

开了 WAL 之后目录里会多两个文件：

```text
-rw-r--r--@ 1 tommywu  wheel    4096 /tmp/dbx/w.db
-rw-r--r--@ 1 tommywu  wheel   32768 /tmp/dbx/w.db-shm
-rw-r--r--@ 1 tommywu  wheel  935272 /tmp/dbx/w.db-wal
```

这是刚写完 2000 行的状态。主库文件只有 4 KB，935 KB 的数据全在 `-wal` 里。跑一次 checkpoint：

```text
-rw-r--r--@ 1 tommywu  wheel  921600 /tmp/dbx/w.db
-rw-r--r--@ 1 tommywu  wheel   32768 /tmp/dbx/w.db-shm
-rw-r--r--@ 1 tommywu  wheel       0 /tmp/dbx/w.db-wal
```

数据回到主库，`-wal` 清零。最后一个连接关闭时，两个文件都会被删掉。

这个中间状态会咬人。我把 2000 行写完之后只拷贝主库文件：

```text
原库行数: 2000
只拷 .db 的副本: OperationalError no such table: t
```

表都不存在。因为 `CREATE TABLE` 那一页也还在 WAL 里。文档专门写了这句：

> The WAL file is part of the persistent state of the database and should be kept with the database if the database is copied or moved.

任何备份、导出、上传日志库的逻辑，只要在 WAL 模式下只处理 `.sqlite` 一个文件，就是错的。要么三个文件一起拷，要么先 `PRAGMA wal_checkpoint(TRUNCATE)`，要么用 SQLite 的 Online Backup API。

默认的自动 checkpoint 阈值是 1000 页，约 4 MB。写得快、读得少的场景下 `-wal` 会一直涨到这个阈值才回写。

WAL 的硬限制也得知道：

> All processes using a database must be on the same host computer; WAL does not work over a network filesystem.

iOS 上这条基本不构成问题，除非你把库放在 iCloud Drive 或者共享容器里。

> 待真机补测：iOS 的文件保护等级（`NSFileProtectionComplete`）会让设备锁屏后数据库文件不可读，此时后台任务里的读写会失败；App 被挂起时若正持有写事务，`-wal` 会残留。这两条在 macOS 上无法复现，需要真机跑一次。SQLite 引擎本身在 iOS 和 macOS 上是同一套实现，上面所有关于事务、索引、WAL 的结论不受平台影响。

---

## 四、SQLITE_BUSY 是怎么来的

四个进程，各插 300 行，全部 WAL 模式，不设 `busy_timeout`：

```text
  进程 3: 成功  29  SQLITE_BUSY 271
  进程 1: 成功 139  SQLITE_BUSY 161
  进程 0: 成功  52  SQLITE_BUSY 248
  进程 2: 成功  43  SQLITE_BUSY 257
  实际落库 263 行 / 期望 1200 行
```

丢了 78%。而且丢得静悄悄。`sqlite3_exec` 返回了 5，如果代码不检查返回值，什么都不会发生。

加一行 `sqlite3_busy_timeout(db, 5000)`：

```text
  进程 0~3: 各成功 300  SQLITE_BUSY 0
  实际落库 1200 行 / 期望 1200 行
```

一行代码的事。**没有 busy handler 的 SQLite 在并发写下会静默丢数据，这是我见过的 iOS 数据库 bug 里最常见的一类。**

但 `busy_timeout` 不是万能的。有一种 `SQLITE_BUSY` 它救不了：

```text
BEGIN DEFERRED  busy_timeout=0      A 升级写 rc=5  database is locked   等了 0.0 ms
BEGIN DEFERRED  busy_timeout=3000   A 升级写 rc=5  database is locked   等了 0.0 ms
BEGIN IMMEDIATE busy_timeout=3000   A 升级写 rc=0  ok
                                    B 那句 INSERT 自己等了 3039.6 ms
```

第二行：设了 3 秒超时，实际 0.0 毫秒就返回了 BUSY。busy handler 根本没被调用。

场景是这样的：A 用 `BEGIN DEFERRED` 开事务，先做了一次 SELECT，于是拿到一个读快照。这期间 B 提交了一次写。现在 A 想写，就要从读事务升级成写事务，可是它手上的快照已经过时了。文档写得很清楚：

> If a write statement occurs while a read transaction is active, then the read transaction is upgraded to a write transaction if possible. If some other database connection has already modified the database or is already in the process of modifying the database, then upgrading to a write transaction is not possible and the write statement will fail with SQLITE_BUSY.

再等下去也没用，因为快照不会变新。SQLite 干脆立刻返回，让调用方回滚重来。

第三行是修法：`BEGIN IMMEDIATE`，开事务时就把写锁拿到手。这时候被挡的是 B，而 B 的 `busy_timeout` 正常工作，老老实实等了 3039 毫秒。

我自己的规矩：**只要一个事务里会有写操作，就用 `BEGIN IMMEDIATE`，不用默认的 DEFERRED。** FMDB 的 `inTransaction:` 用的是 `begin exclusive transaction`（`FMDatabase.m:1150`），WAL 下 EXCLUSIVE 和 IMMEDIATE 等价，所以它默认就避开了这个坑。

---

## 五、FMDB 的并发模型

FMDB 头文件里的警告只有一句：

> @warning Do not instantiate a single `FMDatabase` object and use it across multiple threads. Instead, use `FMDatabaseQueue`.

违反了会怎样？8 个线程共用一个 `FMDatabase`，各插 200 行，跑三轮：

```text
A  第 1 轮：进程被信号 11 (Segmentation fault: 11) 终止，落库   0 行 / 期望 1600
A  第 2 轮：进程被信号 11 (Segmentation fault: 11) 终止，落库 284 行 / 期望 1600
A  第 3 轮：进程被信号 11 (Segmentation fault: 11) 终止，落库 350 行 / 期望 1600
```

三轮全崩。落库行数每次都不一样，还有一轮拿到的是 Bus error 10。这不是数据错乱，是内存直接踩坏。

根因在 SQLite 的编译选项。macOS 系统自带的 libsqlite3 是这么编的：

```text
sqlite3_threadsafe() = 2
```

返回 2 表示 Multi-thread 模式：库本身线程安全，但单个连接不能被多个线程同时使用。FMDB 打开数据库时没有传 `SQLITE_OPEN_FULLMUTEX`，所以一个 `sqlite3*` 上并发调 `sqlite3_step` 就是未定义行为。

`FMDatabaseQueue` 的解法很朴素。`FMDatabaseQueue.m:102`：

```objc
_queue = dispatch_queue_create([[NSString stringWithFormat:@"fmdb.%@", self] UTF8String], NULL);
dispatch_queue_set_specific(_queue, kDispatchQueueSpecificKey, (__bridge void *)self, NULL);
```

第二个参数传 `NULL`，是一个串行队列。所有数据库操作都 `dispatch_sync` 到它上面（`FMDatabaseQueue.m:192`）：

```objc
FMDBRetain(self);

dispatch_sync(_queue, ^() {
    FMDatabase *db = [self database];
    block(db);
    ...
});
```

整个类就干一件事。把并发访问排成队列，同一时刻只有一个线程在碰那个 `sqlite3*`。

三种用法的实测对照（8 线程 × 200 行）：

```text
A  共用一个 FMDatabase       崩溃，落库 0~350 行
B  FMDatabaseQueue           落库 1600 行，耗时  67 ms
C  每线程一个 FMDatabase     落库 1600 行，耗时 195 ms
```

C 也是对的。每个线程开自己的连接，走的是多连接并发那条路，靠 `busy_handler` 化解冲突。FMDB 的 `maxBusyRetryTimeInterval` 默认是 2 秒（`FMDatabase.m:86`），并在 `FMDatabase.m:333` 装了 `sqlite3_busy_handler`，所以第四节那种静默丢数据在 FMDB 上不会发生。

C 比 B 慢 3 倍，因为多连接之间要真的争锁，而串行队列是纯内存排队。多写少读的场景下，一个串行队列反而是更快的方案。

### 嵌套调用：今天已经不是死锁了

流传很广的说法是"`FMDatabaseQueue` 里嵌套调用 `inDatabase:` 会死锁"。我用子进程加 `alarm(3)` 保护跑了一次，两种构建都试了。

Debug 构建（未定义 `NDEBUG`）：

```text
Assertion failed: (currentSyncQueue != self && "inDatabase: was called reentrantly on the same queue, which would lead to a deadlock"),
function -[FMDatabaseQueue inDatabase:], file FMDatabaseQueue.m, line 187.
子进程被信号 6 (Abort trap: 6) 终止
```

这是 FMDB 自己的断言，用 `dispatch_get_specific` 检测出当前就在自己的队列上（`FMDatabaseQueue.m:186-187`）。

Release 构建（`-DNDEBUG`，断言被编掉了）：

```text
子进程被信号 5 (Trace/BPT trap: 5) 终止

崩溃报告 asi 字段：
libdispatch.dylib: "BUG IN CLIENT OF LIBDISPATCH: dispatch_sync called on queue already owned by current thread"
```

libdispatch 自己抓住了。现代 libdispatch 会检测 `dispatch_sync` 到当前线程已经持有的队列，直接 trap，不会真的挂死。

所以"嵌套会死锁"这个说法今天需要改一个词：**嵌套会立刻崩溃，不会挂起。** 对调试来说这是好事，崩溃点就在现场。要在一个事务里做多件事，用 `inTransaction:` 的那个 block，不要在 block 里再调 `inDatabase:`。

### FMDB 的开销

同一个表插 10000 条，全在事务里，7 轮取中位：

```text
裸 sqlite3 C API（预编译语句复用）          4.5 ms   1.0x
FMDB inTransaction + executeUpdate         28.9 ms   6.4x
```

6.4 倍看着不少。我先怀疑 prepare。每次 `executeUpdate:` 都重新编译一遍语句，打开语句缓存试了试：

```text
executeUpdate:  shouldCacheStatements=NO      31.1 ms
executeUpdate:  shouldCacheStatements=YES     26.0 ms
同一队列里直接用 sqlite3 C API 预编译语句      5.4 ms
```

只快了 16%。所以瓶颈不在 prepare，在 ObjC 那一层：`NSString stringWithFormat:` 造字符串、`NSNumber` 装箱、可变参数解析、`id` 到 `sqlite3_bind_*` 的类型分发。语句缓存能省的那点在这些开销面前不显眼。

读的一侧完全是另一个故事：

```text
读出全部 10000 行并访问 name 列
裸 sqlite3，直接用 C 字符串         1.8 ms   1.0x
裸 sqlite3，每行转成 NSString       4.3 ms   2.5x
FMDB stringForColumnIndex:          4.1 ms   2.3x
```

FMDB 读一行的成本和"自己写 `sqlite3_column_text` 再转 NSString"完全一样，甚至略快。读这条路上 FMDB 是零成本抽象。

那 2.5 倍是 `NSString` 的成本，跟 FMDB 无关。你不用 FMDB，只要你的数据最终要变成 `NSString`，这笔钱一样得付。

---

## 六、Core Data 不是数据库

这一节想证明一件事：Core Data 是对象图管理框架，SQLite 只是它可选的后端之一。

同一个 `NSManagedObjectModel`，同一套 API，只换 store 类型：

```text
SQLite     存 3 条读回 3 条  文件 24576 B  前 16 字节: SQLite format 3.
Binary     存 3 条读回 3 条  文件  1755 B  前 16 字节: CoreData........
InMemory   存 3 条读回 3 条  没有落盘文件
```

三种 store，业务代码一行没改。SQLite 那个文件的魔数是 `SQLite format 3`，Binary 那个是 `CoreData` 开头的私有格式，InMemory 压根不落盘。这就是"persistent store"这个抽象层的意义。

### 它生成的表长什么样

用 Core Data 建一个含 `Person`（name / city / age）和 `Note`（text）、两者一对多的 store，插 10000 条，然后直接用 `sqlite3` 命令行打开它：

```text
$ sqlite3 cd.sqlite .schema
CREATE TABLE ZNOTE ( Z_PK INTEGER PRIMARY KEY, Z_ENT INTEGER, Z_OPT INTEGER, ZOWNER INTEGER, ZTEXT VARCHAR );
CREATE TABLE ZPERSON ( Z_PK INTEGER PRIMARY KEY, Z_ENT INTEGER, Z_OPT INTEGER, ZAGE INTEGER, ZCITY VARCHAR, ZNAME VARCHAR );
CREATE INDEX ZNOTE_ZOWNER_INDEX ON ZNOTE (ZOWNER);
CREATE TABLE Z_PRIMARYKEY (Z_ENT INTEGER PRIMARY KEY, Z_NAME VARCHAR, Z_SUPER INTEGER, Z_MAX INTEGER);
CREATE TABLE Z_METADATA (Z_VERSION INTEGER PRIMARY KEY, Z_UUID VARCHAR(255), Z_PLIST BLOB);
CREATE TABLE Z_MODELCACHE (Z_CONTENT BLOB);
```

逐个说：

- `Z_PK`：主键，就是 rowid。Core Data 的 `NSManagedObjectID` 里那个 `p1`、`p2` 就是它。
- `Z_ENT`：实体 ID，指向 `Z_PRIMARYKEY` 表。继承体系里的子类实体和父类共用一张表，靠这一列区分。
- `Z_OPT`：乐观锁版本号。实测每 save 一次涨 1：

```text
第一次 save：  Z_PK=1 Z_ENT=1 Z_OPT=1 ZN=v1
改一次再 save：Z_PK=1 Z_ENT=1 Z_OPT=2 ZN=v2
再改一次：     Z_PK=1 Z_ENT=1 Z_OPT=3 ZN=v3
```

`NSMergeConflict` 那套冲突检测就靠这一列。

- `Z_PRIMARYKEY`：实体名到 `Z_ENT` 的映射，外加一个 `Z_MAX` 高水位：

```text
Z_ENT | Z_NAME | Z_SUPER | Z_MAX
    1 | Note   |       0 |     0
    2 | Person |       0 | 10000
```

Core Data 分配主键不靠 SQLite 的 autoincrement，自己维护 `Z_MAX`，一次批量预留一段。
- `Z_METADATA`：里面的 `Z_PLIST` 是一个 blob，装着模型的版本哈希。迁移时靠它判断当前文件对应哪个模型版本。
- `Z_MODELCACHE`：整个模型的缓存。

再看数据：

```text
Z_PK | Z_ENT | Z_OPT | ZNAME        | ZCITY      | ZAGE
   1 |     2 |     1 | person009164 | Hangzhou   | 62
   2 |     2 |     1 | person005203 | Shenzhen   | 61
   3 |     2 |     1 | person002066 | Wuhan      | 44
```

我按 `person000000` 到 `person009999` 的顺序插入的，落到表里 `Z_PK=1` 的却是 `person009164`。Core Data 不保证插入顺序，因为 context 内部是用集合管理未保存对象的。要有序，模型里就得有一个显式的排序字段。

两件事值得单独拎出来：

`ZNOTE_ZOWNER_INDEX` 是唯一被自动创建的索引，建在关系的外键列上。`ZNAME`、`ZCITY`、`ZAGE` 一个索引都没有。Core Data 只给关系建索引，普通属性要自己在模型编辑器里勾 Index（或者用 `NSFetchIndexDescription`）。第二节测过，10 万行没索引的等值查询是 4 毫秒级。线上再乘几个数量级，就是可感知的卡顿。

关系是一列外键 `ZOWNER`，不是中间表。多对多才会生成 `Z_2NOTES` 那种连接表。

Core Data 的 SQLite store 默认就是 WAL 模式，我实测 `PRAGMA journal_mode` 返回 `wal`。所以第三节那条"只拷 `.sqlite` 会丢数据"对 Core Data 同样适用，而且踩的人更多。很多人根本不知道自己在用 SQLite。

---

## 七、Core Data 的代价，量化

同一张表、同样 10000 条、同样 WAL、同样一个事务或一次 save，7 轮取中位：

```text
裸 sqlite3 C API（预编译语句复用）    4.5 ms   1.0x
FMDB inTransaction + executeUpdate   28.9 ms   6.4x
Core Data（私有队列 context）        60.5 ms  13.3x   建对象 15.3 ms / save 44.9 ms

落盘体积（.sqlite + -wal + -shm）
  裸 SQLite    344064 B
  FMDB         344064 B
  Core Data    788648 B
```

写慢 13 倍，体积大 2.3 倍。体积那部分主要来自 `Z_OPT` / `Z_ENT` 两个额外整数列、`Z_MODELCACHE` 里的模型 blob，以及默认更大的初始文件。

读的一侧：

```text
裸 sqlite3，直接用 C 字符串           1.8 ms   1.0x
裸 sqlite3，每行转成 NSString         4.3 ms   2.5x
FMDB stringForColumnIndex:            4.1 ms   2.3x
Core Data 默认 fetch（对象是 fault）  9.5 ms   5.4x
Core Data returnsObjectsAsFaults=NO   7.7 ms   4.4x
```

有意思的是最后两行。默认 fetch 返回的是 fault（占位对象），我以为"不立刻加载数据"会更快，实测反而慢 23%。因为我在循环里访问了每个对象的 `name`，每次访问都触发一次 fulfil，逐个回填。设 `returnsObjectsAsFaults = NO` 让它一次性批量填好，反而省事。

fault 是为"取出来但只用一部分"设计的。要遍历全部结果并读属性，就把它关掉。

### context 的并发模型

`NSManagedObjectContext` 有两种并发类型：`NSMainQueueConcurrencyType` 绑主队列，`NSPrivateQueueConcurrencyType` 自带一个私有串行队列。所有访问必须在对应队列上，靠 `performBlock:` / `performBlockAndWait:` 进去。

这和 `FMDatabaseQueue` 是同一个思路：用串行队列把并发访问排成队。区别是 Core Data 保护的是内存里的对象图，FMDB 保护的是那个 `sqlite3*` 句柄。

跨 context 直接传 `NSManagedObject` 会怎样？我在私有队列 context 里 fetch 出一个对象，把指针带到主线程直接读它的属性：

```text
私有队列 context 取到对象: person009164
在主线程上直接读这个对象的属性……
居然读到了: person009164
```

什么都没发生。这才是最坑的地方。违规访问大多数时候不报错，也不崩溃，只是在你看不见的地方埋着数据竞争。等到线上出现"偶发的数据错乱"，已经无从查起。

加上 `-com.apple.CoreData.ConcurrencyDebug 1` 再跑一次：

```text
CoreData: annotation: Core Data multi-threading assertions enabled.
退出码 133（SIGTRAP）
```

崩溃报告的栈顶：

```text
CoreData  _PFAssertSafeMultiThreadedAccess_impl
CoreData  -[NSManagedObject valueForKey:]
```

`_PFAssertSafeMultiThreadedAccess_impl`。这个开关我的建议是 Debug scheme 里长期打开，成本几乎为零，能把上面那种沉默的违规变成一个明确的断点。

正确的跨 context 姿势是传 `NSManagedObjectID`：

```text
objectID = x-coredata://862C9067-A31D-456E-8023-2D1BFB678CC1/Person/p1
主队列 context 用 objectWithID: 拿到 person009164，两个对象指针相同? 0
```

指针不同。`NSManagedObject` 绑在 context 上，同一条记录在两个 context 里就是两个对象。`NSManagedObjectID` 只是一个 URI，线程安全，可以随便传。

一个细节：`objectWithID:` 拿到的可能是 fault，且不检查记录是否还存在；如果记录已被删除，访问属性时才抛异常。要立刻确认存在性用 `existingObjectWithID:error:`。

---

## 八、我怎么选

先说三条路的边界，然后给我自己的阈值。

裸 SQLite（自己写 C API）只在两种情况下值得：一是极高频的批量写入，比如埋点日志，13 倍的差距在这里是真金白银；二是要用 SQLite 的高级特性，FTS5 全文检索、R-Tree、`ATTACH` 跨库查询、`WITH RECURSIVE`。除此之外，为了省那几毫秒手写一堆 `sqlite3_bind_*`，我不觉得划算。

FMDB 是我的默认选择。它做的三件事恰好是每个人都要做的：串行队列保证线程安全、装好 busy handler、把 `sqlite3_column_*` 包成 ObjC 类型。写入的 6.4 倍开销听着大，但那是 4.5 ms 和 28.9 ms 的差别，在真实 App 里通常淹没在别的开销里。读的一侧是零成本。

Core Data 解决的问题跟前两个不在一个层面：对象图管理、关系维护与级联删除、变更追踪与撤销、模型版本迁移、`NSFetchedResultsController` 驱动 TableView、CloudKit 同步。如果你的数据模型里关系多、要做迁移、要跟 UI 双向绑定，这些东西自己写要写很久，而且写不对。

我自己的阈值，具体到可以直接照着判断：

1. 单表、少于 5 个实体、几乎没有关系、数据量在 10 万行以内：FMDB。上 Core Data 是拿一个对象图框架当 ORM 用，学习成本和调试成本都不合算。
2. 实体超过 5 个、有明确的一对多多对多、需要级联删除、需要模型版本迁移：Core Data。这些正是它的主场。
3. 要做全文检索：裸 SQLite 加 FTS5。Core Data 没有对应能力，硬套 `CONTAINS` 谓词会退化成全表扫描。
4. 写入频率高于每秒 100 次（埋点、IM 消息、传感器数据）：裸 SQLite，自己控制事务批次。把 10000 条攒成一个事务能省下三个数量级。
5. 需要用 `NSFetchedResultsController` 驱动列表增量刷新：Core Data。这个类省下的代码量足以抵消它的性能代价。

不管选哪条路，有三件事是共同的，我认为比选型本身更重要：

- 开 WAL，除非你的数据库要放在网络文件系统上。
- 写路径上一律 `BEGIN IMMEDIATE`，别用默认的 DEFERRED。
- 备份、导出、迁移逻辑里，`.sqlite`、`-wal`、`-shm` 三个文件一起处理，或者先 checkpoint。

最后是一条我不同意的说法。中文圈很流行"Core Data 就是 SQLite 的封装，性能差因为多了一层"。这句话两头都不对。它不是 SQLite 的封装，SQLite 只是它三种 store 之一，第六节的实验直接证明了这点。而它的性能差也不主要来自"多了一层"——13 倍开销的大头是对象图维护、变更追踪、fault 机制，这些是它提供的功能本身的成本。不需要这些功能的人用它，付了钱没拿到货。需要的人自己写，写出来的东西大概率比它慢也比它错得多。

---

## 总结

事务的收益取决于 journal 模式。rollback journal 下不开事务比开事务慢 800~900 倍，WAL 下只慢 43 倍。慢的原因三分之二是 fsync，三分之一是每个事务创建再删除 `-journal` 文件。

索引的收益取决于选择性，不取决于有没有建。10 万行表上同一条等值查询，命中 1 行快 626 倍，命中 1 万行只快 1.1 倍，因为回表的随机 I/O 抵消了收益。改成覆盖索引（只取索引里已有的列）能把 1.1 倍变成 10 倍。看 `EXPLAIN QUERY PLAN` 要看动词：`SEARCH` 是查找，`SCAN ... USING INDEX` 仍然是全扫。

SQLite 默认的 `LIKE` 对 ASCII 大小写不敏感，所以 `LIKE 'prefix%'` 用不上普通索引。建一个 `COLLATE NOCASE` 索引，同一条查询快 670 倍。

不设 `busy_timeout` 时，四进程并发写丢了 78% 的数据，而且完全静默。`BEGIN DEFERRED` 升级成写事务时产生的那种 BUSY，`busy_timeout` 救不了，只能改用 `BEGIN IMMEDIATE`。

`FMDatabaseQueue` 的全部实现就是一个串行队列加 `dispatch_sync`。多线程共用一个 `FMDatabase` 在 macOS 上是段错误而不是数据错乱，因为系统 libsqlite3 编在 `SQLITE_THREADSAFE=2` 模式。嵌套调用 `inDatabase:` 今天不会挂死，Debug 下 FMDB 自己断言，Release 下 libdispatch trap。

Core Data 用 SQLite 时生成的是 `Z_PK` / `Z_ENT` / `Z_OPT` 那套表结构，只给关系列建索引，属性索引要自己勾。写慢 13 倍、读慢 5 倍、体积大 2.3 倍。跨 context 直接传 `NSManagedObject` 平时不报错，打开 `ConcurrencyDebug` 才会当场 trap。

---

## 参考资料

### 官方

- [SQLite — Write-Ahead Logging](https://www.sqlite.org/wal.html)：WAL 的读写并发语义、`-wal` / `-shm` 文件必须随库一起拷贝、checkpoint 阈值、网络文件系统限制
- [SQLite — BEGIN TRANSACTION](https://www.sqlite.org/lang_transaction.html)：隐式事务的定义，DEFERRED / IMMEDIATE / EXCLUSIVE 的差别，以及 DEFERRED 升级失败为什么必然是 `SQLITE_BUSY`
- [SQLite — Query Planner](https://www.sqlite.org/optoverview.html)：LIKE 优化的前提条件、覆盖索引、`EXPLAIN QUERY PLAN` 的读法
- [SQLite — Database File Format](https://www.sqlite.org/fileformat2.html)：页、B-tree 页、freelist
- [Core Data Programming Guide](https://developer.apple.com/documentation/coredata/)：object graph / persistent container / store 三层
- [Using Core Data in the Background](https://developer.apple.com/documentation/coredata/using-core-data-in-the-background)：两种 concurrency type 与 `objectID` 传递

### 一手源码

- [ccgus/fmdb](https://github.com/ccgus/fmdb)（本文读的是 `d3abf74`）：`FMDatabaseQueue.m:102` 建串行队列、`:186-187` 重入断言、`:192` `dispatch_sync`；`FMDatabase.m:86` 默认 2 秒 busy 重试、`:333` 装 busy handler、`:1150` `inTransaction:` 用的是 `begin exclusive transaction`

### 本地

- [[iOS 持久化选型：沙盒、plist、NSUserDefaults 与 Keychain]]
- [[iOS 序列化：NSCoding、NSSecureCoding 与 JSON 三条路]]
- [[iOS GCD：队列不是线程，以及死锁的准确边界]]：串行队列与 `dispatch_sync` 重入，第五节的背景

---

实验环境：macOS 26.5.2（Build 25F84），Apple Silicon，arm64 原生二进制；Apple clang 21.0.0；系统自带 SQLite 3.51.2（`sqlite3` 命令行）/ 3.51.0（`-lsqlite3` 链接进程序时 `sqlite3_libversion()` 报告值）；FMDB 从 GitHub 主线 `d3abf74` 直接编入；Core Data 用 `-framework CoreData`，模型全部用代码构建，不依赖 `.xcdatamodeld`。全部实验都不需要模拟器。

所有耗时数字都是同一组跑 7 轮取中位数，正文里给出的多为区间或中位值。数据文件在 `/tmp` 下（APFS，内建 SSD）；换文件系统或换机器，绝对值一定会变，量级关系应该保持。

SQLite 引擎和 Core Data 在 iOS 与 macOS 上是同一套实现，事务、索引、WAL、`SQLITE_BUSY` 的行为不随平台变化。

> 待真机补测：文件保护等级（`NSFileProtectionComplete` 及其变体）下锁屏后数据库不可读、App 被挂起时写事务的中断行为、后台任务里 `-wal` 残留的处理，这三项是 iOS 专属，macOS 上无法复现。复现方法：把第三、四节的程序编成 iOS target，给 store 文件设 `NSFileProtectionComplete`，锁屏后用 `BGProcessingTask` 触发一次读写。
