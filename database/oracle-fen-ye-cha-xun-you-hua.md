# \[Oracle] 分页查询优化

## 事情是这样的, 在项目使用中我们经常需要用到分页查询的功能, 但是Oracle直接分页查询在存在排序的情况下会出现分页错乱的情况

> 怎么解决呢??

```sql
-- 原始SQL
SELECT  FROM XXXX WHERE A='B' ORDER BY ID LIMIT 0, 100
```

```sql
-- 改造后
select * from (select oracleB.*, rownum rowno from (
    -- 这里其实就是把原始SQL的分页去掉, 只保留排序, 让分页在外面
    SELECT  FROM XXXX WHERE A='B' ORDER BY ID
) oracleB where rownum <= 100) oracleC where oracleC.rowno > 0
```
