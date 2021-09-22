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
  KEY `idx_age` (`col2`)
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

其中 Node 的部分只存 primary key 值, 而 Leaf Node 的部分是存 row 的所有值
