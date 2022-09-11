# limit优化几点想法

**背景字段：**

- 主键：id
- 索引：KEY idx_name (name),  KEY idx_update_time (update_time) 

**常规分页语句:**

~~~sql
select id,name,balance from account where update_time> '2020-09-19' limit 100000,10;
~~~

该语句使用了普通索引：`update_time`,但是该种索引只会查询所有符合条件的update_time字段，然后将所有符合条件的字段至主键索引树上进行**回表**操作，找出满足条件的行字段，并进行展示，即需要扫描前100010行，取最后10行，所以回表次数会非常多

**优化思路：**

**1.将查询的字段落到索引上**，如果该语句被大量使用，可考虑该方法，将name和balance均加上索引，若不大量使用，加上索引反而会浪费性能（索引需要占内存，不易加过多）

**2.将查询的条件落到主键索引上**

**示例语句：**

~~~sql
select id,name,balance FROM account where id >= (select a.id from account a where a.update_time >= '2020-09-19' limit 100000, 1) AND update_time >= '2020-09-19' LIMIT 10;
-- 建议将update_time作为create_time理解，否则该语句逻辑是错误的
~~~

这种思路是，通过普通索引update_time查询时，只拿id(无需回表)，再通过主键索引树查询需要的数据

**3.INNER JOIN 延迟关联**

**示例语句：**

~~~sql
SELECT  acct1.id,acct1.name,acct1.balance FROM account acct1 INNER JOIN (SELECT a.id FROM account a WHERE a.update_time >= '2020-09-19' ORDER BY a.update_time LIMIT 100000, 10) AS  acct2 on acct1.id= acct2.id;
~~~

与子查询类似，先通过update_time二级索引树查询到满足条件的主键ID，再与原表通过主键ID内连接，这样后面直接走了主键索引了，同时也减少了回表

**4.标签记录法**

记录自己排序到哪一页了，下次排序直接从这一页开始，也能减少回表次数，但是这种方法局限性太大

~~~sql
select  id,name,balance FROM account where id > 100000 order by id limit 10;
~~~

使用使用between...and...也能达到类似效果

~~~sql
select  id,name,balance FROM account where id between 100000 and 100010 order by id desc;
~~~

