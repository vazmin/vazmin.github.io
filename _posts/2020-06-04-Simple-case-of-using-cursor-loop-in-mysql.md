---
title:  在mysql中简单使用游标遍历数据
date:   2020-06-04 21:55:55 +0800
categories:
  - mysql
tags:
  - cursor
---

有时我们需要书写sql语句实现有一点点的复杂的逻辑修复数据，直接在应用层写脚本又略麻烦，那么我们可以使用[Compound Statements](https://dev.mysql.com/doc/refman/5.7/en/sql-compound-statements.html)来进行流程控制修复数据。

## Case

现有张email_xxx表，部分字段如下表格，需将同一email下create_time最大的记录locked设为0，剩余记录设为 1。

| id  | email            | create_time         | locked |
|:---:|:-----------------|:--------------------|-------:|
| 263 | 0c9ad654@foo.bar | 2020-04-02 17:41:02 |      0 |
| 264 | 0c9ad654@foo.bar | 2020-04-02 17:52:34 |      0 |
| 265 | 0c9ad654@foo.bar | 2020-04-02 18:09:18 |      0 |
| 266 | 0c9ad654@foo.bar | 2020-04-02 18:28:59 |      0 |
| 267 | 0c9ad654@foo.bar | 2020-04-02 18:39:35 |      0 |
| 268 | 0c9ad654@foo.bar | 2020-04-02 18:51:43 |      0 |
| 234 | 12adfa6c@foo.bar | 2020-01-07 11:04:57 |      0 |
| 247 | 12adfa6c@foo.bar | 2020-03-13 15:39:16 |      0 |
| 219 | 16c1b318@foo.bar | 2019-12-19 11:54:39 |      0 |
| 253 | 16c1b318@foo.bar | 2020-03-18 16:28:38 |      0 |
| 274 | 46ac9f06@foo.bar | 2020-04-10 18:30:11 |      0 |
| 275 | 46ac9f06@foo.bar | 2020-04-13 14:50:23 |      0 |
| 276 | 46ac9f06@foo.bar | 2020-04-13 16:33:09 |      0 |
| 217 | 9b8c4a14@foo.bar | 2019-12-17 12:22:27 |      0 |

一般做法

```sql
UPDATE user_repeat 
SET locked = 1 
WHERE
 id IN (
 SELECT
  id 
 FROM
  ( SELECT * FROM user_repeat ) a
  LEFT JOIN ( SELECT email, MAX( create_time ) max_create_time 
    FROM user_repeat GROUP BY email ) b ON a.email = b.email 
 WHERE
  a.create_time != b.max_create_time 
 )
```


下面使用复合语句


```sql
DROP PROCEDURE if exists curdemo ;

delimiter //
CREATE PROCEDURE curdemo()
BEGIN
    -- 声明临时变量
    DECLARE temp_email VARCHAR(255);
    DECLARE temp_time TIMESTAMP;
    -- loop flag
    DECLARE done TINYINT DEFAULT 0;
    -- 查出每个email对应的最大时间
    DECLARE cur1 CURSOR FOR 
      SELECT email, MAX(create_time) max_create_time
        FROM email_xxx GROUP BY email;
    -- 如果游标滑到最后，done
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = 1;
        
    OPEN cur1;
        read_loop: LOOP
            -- 从cur1中取一条记录赋值到临时变量
            FETCH FROM cur1 INTO temp_email, temp_time;
            -- 如果没有可用数据，跳出循环
            IF done THEN LEAVE read_loop; END IF;
            -- 如果符合条件，更新locaked的值
            UPDATE email_xxx SET locked = 1 
              WHERE email = temp_email 
                AND create_time != temp_time;
        END LOOP;
    CLOSE cur1;
END;
//
delimiter ;

-- 执行
call curdemo;

```


Result:

| id  | email            | create_time         | locked |
|:---:|:-----------------|:--------------------|-------:|
| 263 | 0c9ad654@foo.bar | 2020-04-02 17:41:02 |      1 |
| 264 | 0c9ad654@foo.bar | 2020-04-02 17:52:34 |      1 |
| 265 | 0c9ad654@foo.bar | 2020-04-02 18:09:18 |      1 |
| 266 | 0c9ad654@foo.bar | 2020-04-02 18:28:59 |      1 |
| 267 | 0c9ad654@foo.bar | 2020-04-02 18:39:35 |      1 |
| 268 | 0c9ad654@foo.bar | 2020-04-02 18:51:43 |      0 |
| 234 | 12adfa6c@foo.bar | 2020-01-07 11:04:57 |      1 |
| 247 | 12adfa6c@foo.bar | 2020-03-13 15:39:16 |      0 |
| 219 | 16c1b318@foo.bar | 2019-12-19 11:54:39 |      1 |
| 253 | 16c1b318@foo.bar | 2020-03-18 16:28:38 |      0 |
| 274 | 46ac9f06@foo.bar | 2020-04-10 18:30:11 |      1 |
| 275 | 46ac9f06@foo.bar | 2020-04-13 14:50:23 |      1 |
| 276 | 46ac9f06@foo.bar | 2020-04-13 16:33:09 |      0 |
| 217 | 9b8c4a14@foo.bar | 2019-12-17 12:22:27 |      0 |




