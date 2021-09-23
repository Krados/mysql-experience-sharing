# MySQL experience sharing [中文](https://github.com/Krados/mysql-experience-sharing/blob/master/README_zh-TW.md)
There is a lot of problem occurred when I use MySQL, so I decided to write this, so when the time goes by I can still remember all of this.


## My Environment
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

## B+ tree data structure
*(This is very IMPORTANT data structure for innodb)*


At the beginning I thought I don't need to understand how B+ tree works, I thought MySQL just a tool like a smartphone I just use it whatever I want, and turns out that I am totally wrong.


If you have no idea about B+ tree go check it out [WIKI](https://zh.wikipedia.org/wiki/B%2B%E6%A0%91)


### When MySQL use B+ tree?
Let's check this table below:
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

don't worry I already visualize this B+ tree, but clustered index only
![GitHub Logo](https://github.com/Krados/mysql-experience-sharing/blob/master/test_table_clustered_index.png)
*pic 1*


Node store PRIMARY KEY value only, but Leaf Node store the whole row's data ,you might want to keep this in mind


## Index
First I want to explain why we need an Index, imagine you have a book you have some topic that you want to see, 
so you can start from page 1 and all the way to the topic you are looking for, or you check the book's table of contents, 
find the topic's page number and go there instantly.


It's not hard to tell which solution is more efficient.


**I'm not gonna explain FULLTEXT INDEX, SPATIAL INDEX in this page.**
