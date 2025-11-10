---
date: 2022-03-21
tags: [sql]
---

# 数据库优化总结

这里总结了从数据库设计和sql结构两个方面的优化

## 数据库设计范式 Normal Form

范式是在数据库的设计时总结出来的一套规范，通常满足3范式就足够了。但由于3范式也存在问题，所以有了3.5范式

范式从1到3是越来越严格，所以满足1NF不一定满足2NF，但满足2NF就肯定满足1NF    

满足所有范式的表没有冗余数据，但也有问题，比如效率不高。这就有了反范式做法，即允许数据冗余，用空间换时间

### 1NF：原子化，单元格里的值不能被分割

|Employee Id| Employee Name | Phone Number | Salary |
|-- | -- | -- | -- |
|1EDU001 | Alex | +91 85532065 <br> +91 85532066 | 60,131 |
|1EDU002 | Alex | +91 85532066 | 75,331 |
|1EDU003 | Alex | +91 85532068 | 63,231 |
|1EDU004 | Alex | +91 85532069 | 70,231 |
|1EDU005 | Alex | +91 85543065 <br> +91 85543075 | 60,531 |

可见，有条目的电话格里有两个号码，是可以分割的， 所以不满足1NF。应改为

|Employee Id| Employee Name | Phone Number | Salary |
|-- | -- | -- | -- |
|1EDU001 | Alex | +91 85532065 | 60,131 |
|1EDU001 | Alex | +91 85532066 | 60,131 |
|1EDU002 | Alex | +91 85532066 | 75,331 |
|1EDU003 | Alex | +91 85532068 | 63,231 |
|1EDU004 | Alex | +91 85532069 | 70,231 |
|1EDU005 | Alex | +91 85543065 | 60,531 |
|1EDU005 | Alex | +91 85543075 | 60,531 |

当所有单元格都无法分割了，就满足了第一范式

### 2NF：属性不允许部分依赖（复合）主键

满足2NF前提要满足1NF, 范式2要求非主属性必须依赖完整复合主键或者主键。  

下表`Employee Id`和`Department Id`组成复合主键  
|Employee Id|Department Id |Office Location |
|---|---|---|
|1EDUO01 | ED-T1 | Pune |
|1EDUO02 | ED-S2 |Bengaluru |
|1EDUO03 | ED-M1 | Delhi |
|1EDUO04 | ED-T3 | Mumbai |
|1EDUO05 | ED-T1 | Pune |


但`Office Location`只依赖主键的`Employee Id`部分。或者说只需`Department Id`就能保证`Office Location`唯一， 所以是部分依赖。这样不满足2NF了，应改成两个表

|Employee Id|Department Id |
|---|---|
|1EDUO01 | ED-T1 |
|1EDUO02 | ED-S2 |
|1EDUO03 | ED-M1 |
|1EDUO04 | ED-T3 |
|1EDUO05 | ED-T1 |

|Department Id |Office Location |
|--|--|
| ED-T1 | Pune |
| ED-S2 |Bengaluru |
| ED-M1 | Delhi |
| ED-T3 | Mumbai |

### 3NF： 依赖不能传递

满足3NF前提要满足2NF, 还有非主属性必须依赖主键/复合主键，不能依赖其他非主属性

Student Id |Student name | Subject Id | Subject | Adress 
---|---|---|---|---
1DT1SENGO1 |Alex | 15CS11|SQL | Goa
1DT1SENGO2 |Barry | 15CS13 | JAVA| Bengaluru
1DT15ENGO3 |Clair | 15CS12 | C++|Delhi
1DT15ENG04 |DavId |15CS13 | JAVA | Kochi

上表的`Subject`依赖非主属性`Subject Id`，而不是依赖`Student Id`， 这就不满足3NF了。应改成

Student Id |Student name | Subject Id  
---|---|---
1DT1SENGO1 |Alex | 15CS11
1DT1SENGO2 |Barry | 15CS13
1DT15ENGO3 |Clair | 15CS12
1DT15ENG04 |DavId |15CS13

Subject Id | Subject | Adress 
---|---|---
15CS11|SQL | Goa
15CS13 | JAVA| Bengaluru
15CS12 | C++|Delhi
15CS13 | JAVA | Kochi

#### 不满足3范式的问题

不满足3范式的数据库会存在以下问题:
* 插入问题： 插入数据时，如果只有部分数据，可能导致插入失败（若某列为NOT NULL）。举例2NF里原来的表，若只有`Department Id`和`Office Location`，就无法插入
* 更新问题： 当更新某个字段时，需要更新其它列。举例2NF的表，当更新Pune时， 第1列和第4列都要更新
* 删除问题： 当删除某些属性时，其他属性也要删除， 举例3NF原来的表，当删除SQL课程时，学生Alex的信息也会丢失

### 3.5NF： BCNF

3.5范式用于解决3范式遗留问题  
例如

Student Id | Subject | Professor
---| --- | ---
1DT15ENG01 | SQL | Prof. Mishra
1DT15ENG02 | JAVA | Prof. Anand 
1DT15ENG02 | C++ | Prof. Kanthi
1DT15ENG03 | JAVA | Prof. Anand
1DT15ENG04 | DBMS | Prof. Lokesh

说明：
* 一个学生可以登记多个科目
* 多个老师可以教授同一个科目
* 每个科目会有一个老师分配给一个学生

所以这个表满足3范式，因为`Student Id`和`Subject`组成了复合主键，决定了`Professor`。  

但是不满足3.5NF，因为`Student Id`和`Subject`组成了复合主键，所以`Subject`是主属性， 而存在`Professor`依赖`Subject`， 虽然`Subject`是主属性，但`Professor`是非主属性，所以不满足BCNF。

此表应改成

Student Id | Professor Id
---| --- 
1DT15ENG01 | 1DTPF01 
1DT15ENG02 | 1DTPF02
1DT15ENG02 | 1DTPF03 


Professor ID | Professor | Subject
---| --- | --- 
1DTPF01 | Prof. Mishra | SQL
1DTPF02 | Prof. Anand | JAVA
1DTPF03 | Prof. Kanthi | C++

### 反范式

没有冗余数据的设计未必是好设计，有时为了效率，必须降低范式标准，适当允许保留冗余数据，以达到用空间换时间的目的。

例如订单表存在`商品`，`单价`，`数量`，`总价`， 这里总价是冗余的，因为可以通过单价和数量来求得，但总价经常需要，所以冗余可以提高效率。

## HAVING 和查询语句优先级

SQL查询字句类似C语言的运算符优先级， 虽然标准里没有定义优先级，但大致为以下
```sql
FROM > WHERE > SELECT > GROUP BY > HAVING > ORDER BY
```
这个顺序可以帮忙理解查询语句是如何执行的，比如因为SELECT优先级低于FROM, 所有只能在FROM中定义alais在SELECT中使用，而不能在SELECT中定义alias并在FROM中使用

HAVING指令通常和GROUP一起用，因为优先级比GROUP低，所以HAVING可以计算出GROUP的值并进行筛选。

```sql
SELECT Sno FROM Grade GROUP BY Sno HAVING COUNT(Cno) > 3;
```
这个语句筛选出相同Sno记录的个数>3的集合

## VIEW和数据冗余

View是一种虚拟表，可以将多表的数据关联在一起，也可以做一些冗余数据，如计算平均值

```sql
CREATE VIEW Cl_avg_age AS SELECT Clno, AVG(Sage) AS avg_age FROM Student;
```

## 事物，存储过程和触发器

存储过程需要提前创建再使用。 因为存储过程中定义变量， 输入输出明确，所以让数据库操作更安全。
```sql
CREATE PROCEDURE ap_returncount @Sno char(7) AS DECLARE @count int SELECT @count = Number FROM Class WHERE Clno = (SELECT Clno FROM Student WHERE Sno = @Sno) RETURN @count;
```

触发器是当触发条件时自动执行的命令，如当插入和更新满足条件时自动执行事物回撤。这是基于事物机制的。
事物可以保证多个操作的原子性， 当莫个操作不满足条件则全部`ROLLBACK`，回撤所有的操作，例如典型的转账场景，必须要扣除和增加都成功时才能成功。

但事物导致效率比较低。

```sql
CREATE TRIGGER tri_ins_upd_student ON Student AFTER INSERT, UPDATE AS IF 40 < (SELECT Number FROM Class WHERE Clno = (SELECT Clno FROM inserted )) ROLLBACK TRANSACTION;
```

## 参考

https://www.edureka.co/blog/normalization-in-sql/