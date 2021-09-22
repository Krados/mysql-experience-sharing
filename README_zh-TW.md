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


### MySQL 有哪些 Index?

#### (聚集索引) clustered index
* 一張表中的 PRIMARY KEY 就會被 MySQL 拿來當 clustered index
* 一張表中如果沒有 PRIMARY KEY 則 MySQL 會拿第一個 NOT NULL UNIQUE index 當 clustered index
* 如果以上兩者皆沒有辦法滿足則 MySQL 會產生一個 hidden index 來當 clustered index

#### (二級索引) secondary index
* 任何非 clustered index 的 index 就稱作 secondary index

