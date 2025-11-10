---
date: 2020-08-28
tags: [sql]
---

# SQL优先级问题

例如数学中的加减乘除，以及编程里的运算符，在sql里也有优先级，依次是：

```
FROM clause
WHERE clause
SELECT clause
GROUP BY clause
HAVING clause
ORDER BY clause
```

优先级依次下降，所以FROM的优先级比WHREE高，这样会有什么影响？ 

## 优先级的问题

因为SELECT的优先级比WHERE低，所以在`SELECT`里面定义的alias别称就不能在`WHERE` 里面使用  
但可以在`ORDER`里面使用，因为ORDER 后执行  


## 常见问题

```
...
GROUP BY a, b, c
ORDER BY NULL
```

```
. . .
GROUP BY a, b, c
ORDER BY a, b, c
```

在上面的两个例子里的ORDER BY都不会执行，分别解释：

1. 第一个例子里的ORDER 是删除GROUP的排序
2. 第二个例子的ORDER是重复了GROUP已经干了的事情

## 验证方法

可以找个在线SQL工具，例如 `http://sqlfiddle.com/`  

先执行创建数据库命令 
```sql
DROP TABLE if exists new_table;

CREATE TABLE `new_table` (
`id` int(11) NOT NULL AUTO_INCREMENT,
`testdecimal` decimal(6,2) DEFAULT NULL,
PRIMARY KEY (`id`));

INSERT INTO `new_table` (`testdecimal`) VALUES ('1234.45');
INSERT INTO `new_table` (`testdecimal`) VALUES ('1234.45');

set @mysqlorder := '';

select @mysqlorder := CONCAT(@mysqlorder," SELECT ") from new_table,(select @mysqlorder := CONCAT(@mysqlorder," FROM ")) tt
JOIN (select @mysqlorder := CONCAT(@mysqlorder," JOIN1 ")) t on ((select @mysqlorder := CONCAT(@mysqlorder," ON1 ")) or rand() < 1)
JOIN (select @mysqlorder := CONCAT(@mysqlorder," JOIN2 ")) t2 on ((select @mysqlorder := CONCAT(@mysqlorder," ON2 ")) or rand() < 1)
where ((select @mysqlorder := CONCAT(@mysqlorder," WHERE ")) or IF(new_table.testdecimal = 1234.45,true,false))
group by (select @mysqlorder := CONCAT(@mysqlorder," GROUPBY ")),id
having (select @mysqlorder := CONCAT(@mysqlorder," HAVING "))
order by (select @mysqlorder := CONCAT(@mysqlorder," ORDERBY "));
```

在执行 `select @mysqlorder;` 结果如下:
```
@mysqlor`der
(null)
```

可见其优先级顺序为
```
FROM JOIN1 JOIN2 WHERE ON2 ON1 ORDERBY GROUPBY SELECT WHERE ON2 ON1 ORDERBY GROUPBY SELECT HAVING HAVING
```
