---
layout: article
title:  "MySQL SELECT语句 一"
categories: mysql
toc: true
ads: true
image:
    teaser: /mysql/select.jpg
---  

[原文连接](https://en.wikibooks.org/wiki/SQL_Exercises/The_computer_store#Sample_dataset)
## 测试数据
{% highlight mysql %}
{% raw %}
CREATE TABLE Manufacturers (
  Code INTEGER,
  Name VARCHAR(255) NOT NULL,
  PRIMARY KEY (Code)   
);

CREATE TABLE Products (
  Code INTEGER,
  Name VARCHAR(255) NOT NULL ,
  Price DECIMAL NOT NULL ,
  Manufacturer INTEGER NOT NULL,
  PRIMARY KEY (Code), 
  FOREIGN KEY (Manufacturer) REFERENCES Manufacturers(Code)
) ENGINE=INNODB;


INSERT INTO Manufacturers(Code,Name) VALUES(1,'Sony');
INSERT INTO Manufacturers(Code,Name) VALUES(2,'Creative Labs');
INSERT INTO Manufacturers(Code,Name) VALUES(3,'Hewlett-Packard');
INSERT INTO Manufacturers(Code,Name) VALUES(4,'Iomega');
INSERT INTO Manufacturers(Code,Name) VALUES(5,'Fujitsu');
INSERT INTO Manufacturers(Code,Name) VALUES(6,'Winchester');

INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(1,'Hard drive',240,5);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(2,'Memory',120,6);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(3,'ZIP drive',150,4);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(4,'Floppy disk',5,6);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(5,'Monitor',240,1);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(6,'DVD drive',180,2);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(7,'CD drive',90,2);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(8,'Printer',270,3);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(9,'Toner cartridge',66,3);
INSERT INTO Products(Code,Name,Price,Manufacturer) VALUES(10,'DVD burner',180,2);
{% endraw %}
{% endhighlight %}
## 练习
{% highlight mysql %}
{% raw %}
1.选择商店中所有产品的名称。
SELECT 
    *
FROM
    Manufacturers;

2.选择商店中所有商品的名称和价格。
SELECT 
    Name, Price
FROM
    Products;


3.选择价格低于或等于200美元的产品名称。
SELECT 
    Name, Price
FROM
    Products a
WHERE
    a.Price <= 200;

4.选择价格在$ 60到$ 120之间的所有产品。
/* whit between and */
SELECT 
    Name, Price
FROM
    Products a
WHERE
    a.Price BETWEEN 60 AND 120;
/* with and */
SELECT 
    Name, Price
FROM
    Products a
WHERE
    a.Price >= 60 AND a.Price <= 120;
5.选择名称和价格（分），即价格必须乘以100。
SELECT 
    Name, Price * 100 AS PriceCents
FROM
    Products;
6.计算所有产品的平均价格。
SELECT 
    AVG(Price) AS avg_price
FROM
    Products;

7.计算制造商代码等于2的所有产品的平均价格。
select avg(Price)  as avg_price from Products a where a.Manufacturer = 2;

8.计算价格大于或等于180美元的产品数量。
select count(*) from Products where Price >= 180;

9.选择价格大于或等于$ 180的所有产品的名称和价格，并按价格（按降序排序），然后按名称（按升序排列）排序。
select  Name, Price from Products where Price >= 180 order by Price desc, Name;

10.选择产品中的所有数据，包括每个产品制造商的所有数据。
select * from Products a , Manufacturers  b where a.Manufacturer = b.Code;
select * from Products a inner join Manufacturers b on a.Manufacturer = b.Code;

11.选择所有产品的产品名称，价格和制造商名称。
select a.Name, a.Price, b.Name from Products a , Manufacturers  b where a.Manufacturer = b.Code;
select a.Name, a.Price, b.Name from Products a join  Manufacturers  b on a.Manufacturer = b.Code;


12.选择每个制造商的产品的平均价格，仅显示制造商的代码。
select avg(Price), Manufacturer from Products group by Manufacturer;

13.选择每个制造商的产品的平均价格，显示制造商的名称。
 /* Without INNER JOIN */
 SELECT AVG(Price), Manufacturers.Name
   FROM Products, Manufacturers
   WHERE Products.Manufacturer = Manufacturers.Code
   GROUP BY Manufacturers.Name;
 
 /* With INNER JOIN */
 SELECT AVG(Price), Manufacturers.Name
   FROM Products INNER JOIN Manufacturers
   ON Products.Manufacturer = Manufacturers.Code
   GROUP BY Manufacturers.Name;

14.选择产品平均价格大于或等于150美元的制造商名称。
-- having字句可以让我们筛选成组后的各种数据，
-- where字句在聚合前先筛选记录，
-- 也就是说作用在group by和having字句前。
-- 而 having子句在聚合后对组记录进行筛选。
SELECT 
    AVG(Price), m.Name
FROM
    Products p
        JOIN
    Manufacturers m ON p.Manufacturer = m.Code
GROUP BY m.name
HAVING AVG(Price) >= 150;

15.选择最便宜商品的名称和价格。
select p.name, p.Price from Products p order by Price asc limit 1;
select p.name, p.Price from Products p where Price = (select min(Price) from Products);

16.选择每个制造商的名称及其最昂贵产品的名称和价格。
 /* With a nested SELECT and without INNER JOIN */
   SELECT A.Name, A.Price, F.Name
   FROM Products A, Manufacturers F
   WHERE A.Manufacturer = F.Code
     AND A.Price =
     (
       SELECT MAX(A.Price)
         FROM Products A
         WHERE A.Manufacturer = F.Code
     );
 
 /* With a nested SELECT and an INNER JOIN */
   SELECT A.Name, A.Price, F.Name
   FROM Products A INNER JOIN Manufacturers F
   ON A.Manufacturer = F.Code
     AND A.Price =
     (
       SELECT MAX(A.Price)
         FROM Products A
         WHERE A.Manufacturer = F.Code
     );


17.添加新产品：扬声器，$ 70，制造商2。
INSERT INTO Products VALUES (11, 'Loudspeakers' , 70 , 2 );

18.将产品8的名称更新为“激光打印机”。
UPDATE Products 
SET 
    Name = 'Laser Printer'
WHERE
    Name = 'Printer';

19.对所有产品应用10％的折扣。
UPDATE Products 
SET 
    Price = price * 0.9;

20.对价格大于或等于120美元的所有产品应用10％的折扣。
UPDATE Products 
SET 
    Price = price * 0.9
WHERE
    Price >= 120;
{% endraw %}
{% endhighlight %}
