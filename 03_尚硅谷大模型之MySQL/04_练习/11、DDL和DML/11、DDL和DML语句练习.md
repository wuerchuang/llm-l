## 第1题

1、创建数据库test01_market

2、创建表格customers

| 字段名    | 数据类型    |
| --------- | ----------- |
| c_num     | int（11）   |
| c_name    | varchar(50) |
| c_contact | varchar(50) |
| c_city    | varchar(50) |
| c_birth   | date        |

**要求3：**将c_contact字段移动到c_birth字段后面

**要求4：**将c_name字段数据类型改为 varchar(70)

**要求5：**将c_contact字段改名为c_phone

**要求6：**增加c_gender字段到c_name后面，数据类型为char(1)

**要求7：**将表名改为customers_info

**要求8：**删除字段c_city

```mysql
#1、创建数据库
CREATE DATABASE test01_market;

#指定对哪个数据库进行操作
USE test01_market;

#2、创建数据表 customers，
CREATE TABLE customers(
	c_num INT ,
	c_name VARCHAR(50),
	c_contact VARCHAR(50),
	c_city VARCHAR(50),
	c_birth DATE
);

#3、将c_contact字段插入到c_birth字段后面
ALTER TABLE customers MODIFY c_contact VARCHAR(50) AFTER c_birth;


#4、将c_name字段数据类型改为 varchar(70).
ALTER TABLE customers MODIFY c_name VARCHAR(70);

#5、将c_contact字段改名为c_phone.
ALTER TABLE customers CHANGE c_contact c_phone VARCHAR(50);


#6、增加c_gender字段，数据类型为char(1)
ALTER TABLE customers ADD c_gender CHAR(1) AFTER c_name;
#默认在最后一列
#加first,加在第一列
#如果要指定在哪列后面,加after 那列的名称


#7、将表名改为customers_info
ALTER TABLE customers RENAME customers_info;


#8、删除字段c_city
ALTER TABLE customers_info DROP c_city ;
```

## 第2题

1、创建数据库test02_library

2、创建表格books

| 字段名  | 字段说明 | 数据类型      | 允许为空 | 唯一 |
| ------- | -------- | ------------- | -------- | ---- |
| b_id    | 书编号   | int(11)       | 否       | 是   |
| b_name  | 书名     | varchar（50） | 否       | 否   |
| authors | 作者     | varchar(100)  | 否       | 否   |
| price   | 价格     | float         | 否       | 否   |
| pubdate | 出版日期 | year          | 否       | 否   |
| note    | 说明     | varchar(100)  | 是       | 否   |
| num     | 库存     | int(11)       | 否       | 否   |

3、向books表中插入记录

1） 指定所有字段名称插入第一条记录

2）不指定字段名称插入第二记录

3）同时插入多条记录（剩下的所有记录）

| b_id | b_name        | authors         | price | pubdate | note     | num  |
| ---- | ------------- | --------------- | ----- | ------- | -------- | ---- |
| 1    | Tal of AAA    | Dickes          | 23    | 1995    | novel    | 11   |
| 2    | EmmaT         | Jane lura       | 35    | 1993    | joke     | 22   |
| 3    | Story of Jane | Jane Tim        | 40    | 2001    | novel    | 0    |
| 4    | Lovey Day     | George Byron    | 20    | 2005    | novel    | 30   |
| 5    | Old land      | Honore Blade    | 30    | 2010    | law      | 0    |
| 6    | The Battle    | Upton Sara      | 30    | 1999    | medicine | 40   |
| 7    | Rose Hood     | Richard haggard | 28    | 2008    | cartoon  | 28   |

4、将小说类型(novel)的书的价格都增加5。

5、将名称为EmmaT的书的价格改为40。

6、删除库存为0的记录

```mysql
#创建数据库test02_library
CREATE DATABASE test02_library;

#指定使用哪个数据库
USE test02_library;

#创建表格books
CREATE TABLE books(
	b_id INT,
	b_name VARCHAR(50),
	`authors` VARCHAR(100),
	price FLOAT,
	pubdate YEAR,
	note VARCHAR(100),
	num INT
);

#指定所有字段名称插入第一条记录
INSERT INTO books (b_id,b_name,`authors`,price,pubdate,note,num)
VALUES(1,'Tal of AAA','Dickes',23,1995,'novel',11);

#不指定字段名称插入第二记录
INSERT INTO books 
VALUE(2,'EmmaT','Jane lura',35,1993,'Joke',22);

#同时插入多条记录（剩下的所有记录）。
INSERT INTO books VALUES
(3,'Story of Jane','Jane Tim',40,2001,'novel',0),
(4,'Lovey Day','George Byron',20,2005,'novel',30),
(5,'Old land','Honore Blade',30,2010,'Law',0),
(6,'The Battle','Upton Sara',30,1999,'medicine',40),
(7,'Rose Hood','Richard haggard',28,2008,'cartoon',28);

#将小说类型(novel)的书的价格都增加5。
UPDATE books SET price=price+5 WHERE note = 'novel';

#将名称为EmmaT的书的价格改为40。
UPDATE books SET price=40 WHERE b_name='EmmaT';

#删除库存为0的记录
DELETE FROM books WHERE num=0;
```



## 第3题

1、创建数据库test03_bookstore

2、创建book表

```mysql
+----------+--------------+------+-----+---------+----------------+
| Field    | Type         | Null | Key | Default | Extra          |
+----------+--------------+------+-----+---------+----------------+
| id       | int(11)      | NO   | PRI | NULL    | auto_increment |
| title    | varchar(100) | NO   |     | NULL    |                |
| author   | varchar(100) | NO   |     | NULL    |                |
| price    | double(11,2) | NO   |     | NULL    |                |
| sales    | int(11)      | NO   |     | NULL    |                |
| stock    | int(11)      | NO   |     | NULL    |                |
| img_path | varchar(100) | NO   |     | NULL    |                |
+----------+--------------+------+-----+---------+----------------+
```

尝试添加部分模拟数据，参考示例如下：

```mysql
+----+-------------+------------+-------+-------+-------+----------------------------+
| id | title       | author     | price | sales | stock | img_path                   |
+----+-------------+------------+-------+-------+-------+-----------------------------+
|  1 | 解忧杂货店    | 东野圭吾   | 27.20 |   102 |    98 | upload/books/解忧杂货店.jpg   |
|  2 | 边城         | 沈从文     | 23.00 |   102 |    98 | upload/books/边城.jpg       |
+----+---------------+------------+-------+-------+-------+----------------------------+
```

3、创建用户表users，并插入数据

```mysql
+----------+--------------+------+-----+---------+----------------+
| Field    | Type         | Null | Key | Default | Extra          |
+----------+--------------+------+-----+---------+----------------+
| id       | int(11)      | NO   | PRI | NULL    | auto_increment |
| username | varchar(100) | NO   | UNI | NULL    |                |
| password | varchar(100) | NO   |     | NULL    |                |
| email    | varchar(100) | YES  |     | NULL    |                |
+----------+--------------+------+-----+---------+----------------+
```

尝试添加部分模拟数据，参考示例如下：使用md5加密函数对密码进行加密

```mysql
+----+----------+----------------------------------+--------------------+
| id | username | password                         | email              |
+----+----------+----------------------------------+--------------------+
|  1 | admin    | e10adc3949ba59abbe56e057f20f883e | admin@atguigu.com  |
+----+----------+----------------------------------+--------------------+
```

4、创建订单表orders

```mysql
+--------------+--------------+------+-----+---------+-------+
| Field        | Type         | Null | Key | Default | Extra |
+--------------+--------------+------+-----+---------+-------+
| id           | varchar(100) | NO   | PRI | NULL    |       |
| order_time   | datetime     | NO   |     | NULL    |       |
| total_count  | int(11)      | NO   |     | NULL    |       |
| total_amount | double(11,2) | NO   |     | NULL    |       |
| state        | int(11)      | NO   |     | NULL    |       |
| user_id      | int(11)      | NO   | MUL | NULL    |       |
+--------------+--------------+------+-----+---------+-------+
```

尝试添加部分模拟数据，参考示例如下：

```mysql
+----------------+---------------------+-------------+--------------+-------+---------+
| id             | order_time          | total_count | total_amount | state | user_id |
+----------------+---------------------+-------------+--------------+-------+---------+
| 15294258455691 | 2018-06-20 00:30:45 |           2 |        50.20 |     0 |       1 |
+----------------+---------------------+-------------+--------------+-------+---------+
```

5、创建订单明细表order_items

```mysql
+----------+--------------+------+-----+---------+----------------+
| Field    | Type         | Null | Key | Default | Extra          |
+----------+--------------+------+-----+---------+----------------+
| id       | int(11)      | NO   | PRI | NULL    | auto_increment |
| count    | int(11)      | NO   |     | NULL    |                |
| amount   | double(11,2) | NO   |     | NULL    |                |
| title    | varchar(100) | NO   |     | NULL    |                |
| author   | varchar(100) | NO   |     | NULL    |                |
| price    | double(11,2) | NO   |     | NULL    |                |
| img_path | varchar(100) | NO   |     | NULL    |                |
| order_id | varchar(100) | NO   | MUL | NULL    |                |
+----------+--------------+------+-----+---------+----------------+
```

尝试添加部分模拟数据，参考示例如下：

```mysql
+----+-------+--------+---------+---------+-------+----------------+----------------+
| id |count| amount| title    | author   | price | img_path       | order_id       |
+----+-------+--------+------------+----------+-------+----------------+----------------+
|  1 |   1 |  27.20| 解忧杂货店 | 东野圭吾 | 27.20 | static/img/default.jpg|15294258455691 |
|  2 |   1 |  23.00| 边城      | 沈从文   | 23.00 | static/img/default.jpg|15294258455691 |
+----+-------+--------+------------+----------+-------+------------+----------------+
```

参考答案：

```mysql
CREATE DATABASE `bookstore`;

USE `bookstore`;

CREATE TABLE `books` (
  `id` INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `title` VARCHAR(100) NOT NULL,
  `author` VARCHAR(100) NOT NULL,
  `price` DOUBLE(11,2) NOT NULL,
  `sales` INT(11) NOT NULL,
  `stock` INT(11) NOT NULL,
  `img_path` VARCHAR(100) NOT NULL
) ;

INSERT  INTO `books`(`id`,`title`,`author`,`price`,`sales`,`stock`,`img_path`) VALUES 
(1,'解忧杂货店','东野圭吾',27.20,102,98,'upload/books/解忧杂货店.jpg'),
(2,'边城','沈从文',23.00,102,98,'upload/books/边城.jpg');


CREATE TABLE `users` (
  `id` INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `username` VARCHAR(100) NOT NULL UNIQUE,
  `password` VARCHAR(100) NOT NULL,
  `email` VARCHAR(100) DEFAULT NULL
) ;


INSERT  INTO `users`(`id`,`username`,`password`,`email`) VALUES 
(1,'admin',MD5('123456'),'admin@atguigu.com');


CREATE TABLE `orders` (
  `id` VARCHAR(100) NOT NULL PRIMARY KEY,
  `order_time` DATETIME NOT NULL,
  `total_count` INT(11) NOT NULL,
  `total_amount` DOUBLE(11,2) NOT NULL,
  `state` INT(11) NOT NULL,
  `user_id` INT(11) NOT NULL,
  FOREIGN KEY (`user_id`) REFERENCES `users` (`id`)
) ;


INSERT  INTO `orders`(`id`,`order_time`,`total_count`,`total_amount`,`state`,`user_id`) 
VALUES ('15294258455691','2018-06-20 00:30:45',2,50.20,0,1);

CREATE TABLE `order_items` (
  `id` INT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `count` INT(11) NOT NULL,
  `amount` DOUBLE(11,2) NOT NULL,
  `title` VARCHAR(100) NOT NULL,
  `author` VARCHAR(100) NOT NULL,
  `price` DOUBLE(11,2) NOT NULL,
  `img_path` VARCHAR(100) NOT NULL,
  `order_id` VARCHAR(100) NOT NULL ,
  FOREIGN KEY (`order_id`) REFERENCES `orders` (`id`)
) ;


INSERT  INTO `order_items`(`id`,`count`,`amount`,`title`,`author`,`price`,`img_path`,`order_id`) 
VALUES (1,1,27.20,'解忧杂货店','东野圭吾',27.20,'static/img/default.jpg','15294258455691'),
(2,1,23.00,'边城','沈从文',23.00,'static/img/default.jpg','15294258455691');
```

