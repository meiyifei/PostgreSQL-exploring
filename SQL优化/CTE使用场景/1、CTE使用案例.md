# 				CTE使用案例

```sql
-- 创建测试表
create table t2(id int,name text);
postgres=# create table t1(id1 int,foreign key(id2) references t2(id),id2 int,foreign key(id2) references t2(id));
CREATE TABLE

-- 如果想要查询的条件是这样的(id1,id2,id1_anme,id2_name)
   t1.id1, t1.id2, id1_name, id2_name where id1=? and id2=?  
   
-- 根据id1连接id1_name
	postgres=# select t1.id1,t1.id2,t2.name id2_name from t1,t2 where t1.id1=t2.id and  t1.id1=45 and t1.id2=896;
 id1 | id2 |  id2_name  
-----+-----+------------
  45 | 896 | cebjkjjkce
(1 row)
-- 根据id2连接,得到id2_name
	postgres=# select t1.id1,t1.id2,t2.name id2_name from t1,t2 where t1.id2=t2.id and  t1.id1=45 and t1.id2=896;
 id1 | id2 | id2_name 
-----+-----+----------
  45 | 896 | ceec
(1 row)

-- 也就是需要将行转换成列

-- 用子查询
postgres=# select t1.id1,t1.id2,t2.name id1_name,(select t2.name  id2_name from t1,t2 where t1.id2=t2.id and  t1.id1=45 and t1.id2=896) from t1,t2 where t1.id1=t2.id and  t1.id1=45 and t1.id2=896;
 id1 | id2 |  id1_name  | id2_name 
-----+-----+------------+----------
  45 | 896 | cebjkjjkce | ceec
(1 row)

-- CTE

postgres=# with tt as (select t1.id1,t1.id2,t2.name id1_name from t1,t2 where t1.id1=t2.id and  t1.id1=45 and t1.id2=896),
tt1 as (select t1.id1,t1.id2,t2.name id2_name from t1,t2 where t1.id2=t2.id and  t1.id1=45 and t1.id2=896)
select tt.id1,tt.id2,tt.id1_name,tt1.id2_name from tt,tt1 where tt.id1=tt1.id1 and tt.id2=tt1.id2;
 id1 | id2 |  id1_name  | id2_name 
-----+-----+------------+----------
  45 | 896 | cebjkjjkce | ceec
(1 row)
```

