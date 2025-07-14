创建表 (CREATE TABLE)
```
CREATE TABLE 表名 (
    列名1 数据类型 [约束],
    列名2 数据类型 [约束],
    ...
    PRIMARY KEY (列名)
);
```

修改表结构 (ALTER TABLE)
```
ALTER TABLE 表名
ADD 列名 数据类型 [约束];
```

删除表 (DROP TABLE)
```
DROP TABLE 表名;
```

数据查询 (SELECT)
```
SELECT 列1, 列2, ...(*)
FROM 表名
WHERE 条件;
```

数据插入 (INSERT)
```
INSERT INTO 表名 (列1, 列2, ...)
VALUES (值1, 值2, ...);
```

数据更新 (UPDATE)
```
UPDATE 表名
SET 列1 = 新值1, 列2 = 新值2, ...
WHERE 条件;
```

数据删除 (DELETE)
```
DELETE FROM 表名
WHERE 条件;
```


聚合函数 (Aggregate Functions)
```
SELECT 函数名(关键字 指定列) AS 新列名 FROM 表名;
```

分组 (GROUP BY)
```
SELECT 列名1, 聚合函数(列名2)
FROM 表名
WHERE 条件
GROUP BY 列名1
HAVING 分组条件
ORDER BY 列名;
```

连接 (JOIN)
```
内连接
SELECT 列名
FROM 表1
INNER JOIN 表2 ON 表1.共同列 = 表2.共同列;
```

```
左连接
SELECT 列名
FROM 表1
LEFT JOIN 表2 ON 表1.共同列 = 表2.共同列;
```

```
右连接
SELECT 列名
FROM 表1
RIGHT JOIN 表2 ON 表1.共同列 = 表2.共同列;
```

```
全连接
SELECT 列名
FROM 表1
FULL JOIN 表2 ON 表1.共同列 = 表2.共同列;
```

视图 (VIEW)
```
CREATE VIEW 视图名称 AS
SELECT 列1, 列2, ...
FROM 表名
WHERE 条件;
```

索引(INDEX)
```
创建索引
CREATE [UNIQUE | FULLTEXT] INDEX 索引名
ON 表名 (列1 [ASC|DESC], 列2 [ASC|DESC], ...);
```

```
删除索引
DROP INDEX 索引名 ON 表名; -- MySQL / SQL Server
-- 或者
DROP INDEX 索引名; -- Oracle / PostgreSQL (如果索引名在数据库中是唯一的)
```