# MySQL 使用經驗分享 [EN](https://github.com/Krados/mysql-experience-sharing/blob/master/README.md)
分享自己再使用 MySQL 碰到的各種問題, 自己做個紀錄以免時間久了甚麼都忘了XD

## 我使用的環境
OS:


Distributor ID:	Ubuntu
Description:	Ubuntu 16.04.7 LTS
Release:	16.04
Codename:	xenial


MySQL:


mysql  Ver 8.0.25 for Linux on x86_64 (MySQL Community Server - GPL)


Storage Engine:


InnoDB (default)


ISOLATION_LEVEL:


REPEATABLE-READ (default)


## B+ tree 數據結構 
*(非常重要!!!是一切的基礎!!!)*


我自己剛開始接觸 MySQL 根本不覺得我需要去理解 B+ tree 的原理, 我是想著 MySQL 就是個工具只要好用就行, 就像我們用著手機好用就行應該不需要去理解它內部是如何運作,
可是事情沒有這麼簡單就像人生一樣.


如果對 B+ tree 一無所知請先去 [WIKI](https://zh.wikipedia.org/wiki/B%2B%E6%A0%91) 惡補一下


### MySQL 在哪裡使用到 B+ tree?
以底下這個例子來說:
```
CREATE TABLE `test_table` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `col1` varchar(255) NOT NULL,
  `col2` tinyint(4) NOT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_col2` (`col2`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4;

INSERT INTO test_table (col1, col2) VALUES('A', 100);
INSERT INTO test_table (col1, col2) VALUES('B', 99);
INSERT INTO test_table (col1, col2) VALUES('C', 98);
INSERT INTO test_table (col1, col2) VALUES('D', 97);
INSERT INTO test_table (col1, col2) VALUES('E', 96);
INSERT INTO test_table (col1, col2) VALUES('F', 95);
INSERT INTO test_table (col1, col2) VALUES('G', 94);
```

視覺化總是最簡單的, 我們來看一下這張表 B+ tree 長怎樣, 這邊只呈現簡單化 clustered index 的 B+ tree
![GitHub Logo](https://github.com/Krados/mysql-experience-sharing/blob/master/test_table_clustered_index.png)
*圖1*


其中 Node 的部分只存 primary key 值, 而 Leaf Node 的部分是存 row 的所有值


## Index (索引)
首先先講解一下為何需要索引, 當我們看一本書我們想要快速的找到某一個我們想看的主題, 我們可以從第 1 頁開始翻翻到直到我們找到為止, 或者我們可以翻到書本最前面的目錄找到主題對應的頁數.


從上面的例子來看應該不難看出哪一種效率比較高, 索引的目標就是提高查找的效率.


**這篇文章會直接跳過 FULLTEXT INDEX, SPATIAL INDEX 不做深入的解釋.**

### MySQL 有哪些 Index?

#### (聚集索引) clustered index
* 一張表中的 PRIMARY KEY 就會被 MySQL 拿來當 clustered index
* 一張表中如果沒有 PRIMARY KEY 則 MySQL 會拿第一個 NOT NULL UNIQUE index 當 clustered index
* 如果以上兩者皆沒有辦法滿足則 MySQL 會產生一個 hidden index 來當 clustered index

#### (二級索引) secondary index
* 任何非 clustered index 的 index 就稱作 secondary index

#### (覆蓋索引) covering index
* covering index 算是一個特別的 index, 它不存在在表中, 是表示再下查詢時不需要做回表的動作, 剛好在 secondary index 取到所有需要的欄位

### 如何使用 MySQL Index
剛開始接觸 MySQL 時我天真的以為只要表上有加上 index 甚麼查詢都會變快...


**雖然 index 可以讓查詢速度變快, 前提是用法要是對的!!!**


完全沒使用到 index 的例子:
```
mysql> EXPLAIN select * from test_table\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: test_table
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 7
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```
這邊看到 key 欄位是 NULL 完全沒使用到 index, type 欄位是 ALL 代表是整張表的掃描效率極低.


使用 PRIMARY KEY 的例子:
```
mysql> EXPLAIN select * from test_table where id = 1\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: test_table
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```
這邊看到 key 欄位是 PRIMARY, type 欄位是 const 代表是針對 PRIMARY KEY 或者 UNIQUE KEY INDEX 做過濾.


使用 index 但會回表的例子:
```
mysql> EXPLAIN select * from test_table where col2 = 100\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: test_table
   partitions: NULL
         type: ref
possible_keys: idx_col2
          key: idx_col2
      key_len: 1
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```
這邊看到 key 欄位是 idx_col2, type 欄位是 ref 通常是指一般使用最左前綴規則的索引查詢.


這邊簡單說明一下最左前綴規則, 假設你有三個欄位 (col1, col2, col3) 組成的 index, 你的 where 條件只能遵從以下規範:
* where col1 = ?
* where col1 = ? and col2 = ?
* where col1 = ? and col2 = ? and col3 = ?
* where col1 (>=, >, <, <=) ?
* where col1 = ? and col2 (>=, >, <, <=) ?
* where col1 = ? and col2 = ? and col3 (>=, >, <, <=) ?


使用 index 但不會回表的例子:
```
mysql> EXPLAIN select id from test_table where col2 = 100\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: test_table
   partitions: NULL
         type: ref
possible_keys: idx_col2
          key: idx_col2
      key_len: 1
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)

mysql> EXPLAIN select col2 from test_table where col2 = 100\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: test_table
   partitions: NULL
         type: ref
possible_keys: idx_col2
          key: idx_col2
      key_len: 1
          ref: const
         rows: 1
     filtered: 100.00
        Extra: Using index
1 row in set, 1 warning (0.00 sec)
```
這邊會發現回表的與否只差在 Extra 欄位一個是 NULL 一個是 Using index, 不回表其實就代表覆蓋索引 covering index, 你需要的欄位在 secondary index 皆有不需要回去 clustered index 做在一次的存取.


這邊上張圖來解講回不回表的差異


![with_index_condition_pushdown](https://github.com/Krados/mysql-experience-sharing/blob/master/with_index_condition_pushdown.png)

*圖2*


如果查詢只需要 id 或 col2 的話不需要回到 clustered index 取額外的值, 因為 secondary index 的 leaf node 有存了 index 值以及 PRIMARY KEY 值(非物理位置地址)


![without_index_condition_pushdown](https://github.com/Krados/mysql-experience-sharing/blob/master/without_index_condition_pushdown.png)

*圖3*


如果查詢不只需要 id 或 col2 則需要回到 clustered index 取值又稱回表, 例如: col1 的值


不同 type 擁有不同的效能, 左至右由快到慢

system > const > eq_ref > ref > range ~ index_merge > index > ALL


## MySQL Lock 如何運作

在我深入理解 lock 運作前我認為以下的查詢不會有任何的 lock 會發生, 但災難其實已經發生只是我不知道XD
```
mysql> begin; update test_table set col1 = 'ABC' where col1 = 'C';
Query OK, 0 rows affected (0.00 sec)

Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```
再講解上面的災難前我先介紹下 InnoDB 底下有哪些 lock.


### 共享及排他鎖
* shared (S)lock 共享鎖允許交易對(列)做讀取的動作
* exclusive (X)lock 排他鎖允許交易對(列)做更新及刪除的動作


舉個例子: 
* 交易 T1 持有列 r 的 (S)lock, 如果交易 T2 也想持有列 r 的 (S)lock, 交易 T2 會立刻擁有列 r 的 (S)lock
* 交易 T1 持有列 r 的 (S)lock, 如果交易 T2 想持有列 r 的 (X)lock 則必須等待交易 T1 釋放 (S)lock


### 意向共享及排他鎖
* intention shared lock (IS) 意向共享鎖表示有交易想要對表 table 底下的列 r 下 (S)lock, 所以會在 table 上標記 (IS)
* intention exclusive lock (IS) 意向排他鎖表示有交易想要對表 table 底下的列 r 下 (X)lock, 所以會在 table 上標記 (IX)

### S(lock) & X(lock) & (IS) & (IX) 的兼容性
 / | X | IX | S | IS
------------ | -------------| -------------| -------------| -------------
X|衝突|衝突|衝突|衝突
IX|衝突|兼容|衝突|兼容
S|衝突|衝突|兼容|兼容
IS|衝突|兼容|兼容|兼容


## MySQL InnoDB Lock type
* record lock: 單一列上的鎖


例子: 假設 id 為 PRIMARY KEY, 執行 SELECT * FROM t WHERE id = 1 FOR UPDATE, 就對 id = 1 的列上了 record (X)lock
* gap lock: 間隙鎖, 鎖定一個範圍, 但不包含紀錄本身


例子: 假設 c1 為 INDEX, 執行 SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE, 當你想寫入 15 至 t.c1 則會被阻擋
* next key lock: gap + record lock, 鎖定一個範圍同時也鎖定紀錄本身(innodb 再大部分的情況下會使用此種 lock type)


例子: 假設一個索引有 10, 11, 13, 20 那麼可能的 next key lock 區間為:
() <- 代表不包含
[] <- 代表包含


(negative infinity, 10]


(10, 11]


(11, 13]


(13, 20]


(20, positive infinity)


以上 3 種 lock type 再接下的 case study 會一一介紹.


## Lock Case Study

### Case1: 意外的全表鎖
上面提到的例子就是一個對 innodb lock 的機制完全不瞭解而意外造成的全表鎖
```
mysql> begin; update test_table set col1 = 'ABC' where col1 = 'C';
Query OK, 0 rows affected (0.00 sec)

Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```
我們來看看 MySQL 怎麼幫我們下鎖的. **請注意 performance_schema.data_locks 這張表是 MySQL 8 後才加入的**
```
mysql> SELECT * FROM performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140096057640008:1167:140096061923888
ENGINE_TRANSACTION_ID: 5415
            THREAD_ID: 67
             EVENT_ID: 89
        OBJECT_SCHEMA: yolo
          OBJECT_NAME: test_table
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 140096061923888
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140096057640008:50:4:1:140096061920976
ENGINE_TRANSACTION_ID: 5415
            THREAD_ID: 67
             EVENT_ID: 89
        OBJECT_SCHEMA: yolo
          OBJECT_NAME: test_table
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140096061920976
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: supremum pseudo-record
*************************** 3. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140096057640008:50:4:5:140096061920976
ENGINE_TRANSACTION_ID: 5415
            THREAD_ID: 67
             EVENT_ID: 89
        OBJECT_SCHEMA: yolo
          OBJECT_NAME: test_table
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140096061920976
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 4
*************************** 4. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140096057640008:50:4:6:140096061920976
ENGINE_TRANSACTION_ID: 5415
            THREAD_ID: 67
             EVENT_ID: 89
        OBJECT_SCHEMA: yolo
          OBJECT_NAME: test_table
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140096061920976
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 5
*************************** 5. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140096057640008:50:4:7:140096061920976
ENGINE_TRANSACTION_ID: 5415
            THREAD_ID: 67
             EVENT_ID: 89
        OBJECT_SCHEMA: yolo
          OBJECT_NAME: test_table
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140096061920976
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 6
*************************** 6. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140096057640008:50:4:8:140096061920976
ENGINE_TRANSACTION_ID: 5415
            THREAD_ID: 67
             EVENT_ID: 89
        OBJECT_SCHEMA: yolo
          OBJECT_NAME: test_table
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140096061920976
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 7
*************************** 7. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140096057640008:50:4:10:140096061920976
ENGINE_TRANSACTION_ID: 5415
            THREAD_ID: 67
             EVENT_ID: 89
        OBJECT_SCHEMA: yolo
          OBJECT_NAME: test_table
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140096061920976
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 1
*************************** 8. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140096057640008:50:4:13:140096061920976
ENGINE_TRANSACTION_ID: 5415
            THREAD_ID: 67
             EVENT_ID: 89
        OBJECT_SCHEMA: yolo
          OBJECT_NAME: test_table
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140096061920976
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 2
*************************** 9. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140096057640008:50:4:15:140096061920976
ENGINE_TRANSACTION_ID: 5415
            THREAD_ID: 67
             EVENT_ID: 89
        OBJECT_SCHEMA: yolo
          OBJECT_NAME: test_table
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140096061920976
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 3
9 rows in set (0.00 sec)
```
先來看一下意向鎖怎麼下的
```
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140096057640008:1167:140096061923888
ENGINE_TRANSACTION_ID: 5415
            THREAD_ID: 67
             EVENT_ID: 89
        OBJECT_SCHEMA: yolo
          OBJECT_NAME: test_table
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 140096061923888
            LOCK_TYPE: TABLE
            LOCK_MODE: IX
          LOCK_STATUS: GRANTED
```
因為我們要對 test_table 的列做更新 MySQL 幫我們在 test_table 上了一個 (IX) 的表級鎖


再來看一下 row 3~9 的資料
```
*************************** 3. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140096057640008:50:4:5:140096061920976
ENGINE_TRANSACTION_ID: 5415
            THREAD_ID: 67
             EVENT_ID: 89
        OBJECT_SCHEMA: yolo
          OBJECT_NAME: test_table
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140096061920976
            LOCK_TYPE: RECORD
            LOCK_MODE: X
          LOCK_STATUS: GRANTED
            LOCK_DATA: 4

ignore others cuz its too much
```
會發現 MySQL 幫我們在每一筆紀錄上了一個 (X)lock record lock, 但基本上已經把全部的 record 上了 (X)lock, 這樣就造成了意外的全表鎖
