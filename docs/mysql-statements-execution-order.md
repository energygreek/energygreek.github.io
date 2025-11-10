---
date: 2019-01-10
tags:
  - mysql
---

# mysql sql语句优先级

```
The actual execution of MySQL statements is a bit tricky. However, the standard does specify the order of interpretation of elements in the query. This is basically in the order that you specify, although I think HAVING and GROUP BY could come after SELECT:

    FROM clause
    WHERE clause
    SELECT clause
    GROUP BY clause
    HAVING clause
    ORDER BY clause

This is important for understanding how queries are parsed. You cannot use a column alias defined in a SELECT in the WHERE clause, for instance, because the WHERE is parsed before the SELECT. On the other hand, such an alias can be in the ORDER BY clause.

As for actual execution, that is really left up to the optimizer. For instance:

. . .
GROUP BY a, b, c
ORDER BY NULL

and

. . .
GROUP BY a, b, c
ORDER BY a, b, c

both have the effect of the ORDER BY not being executed at all -- and so not executed after the GROUP BY (in the first case, the effect is to remove sorting from the GROUP BY and in the second the effect is to do nothing more than the GROUP BY already does).

```


## 引用

https://stackoverflow.com/questions/24127932/mysql-query-clause-execution-order
