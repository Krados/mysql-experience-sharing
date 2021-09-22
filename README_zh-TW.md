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

如果查詢只需要 id 或 col2 的話不需要回到 clustered index 取額外的值, 因為 secondary index 的 leaf node 有存了 index 值以及 PRIMARY KEY 值(非物理位置地址)


![without_index_condition_pushdown](https://github.com/Krados/mysql-experience-sharing/blob/master/without_index_condition_pushdown.png)


如果查詢不只需要 id 或 col2 則需要回到 clustered index 取值又稱回表, 例如: col1 的值
