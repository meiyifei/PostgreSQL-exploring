# 		递归CTE使用的常见场景

### 一、求表某一列的唯一值

```sql
-- 表结构

test=# \d+ test;
                                    Table "public.test"
 Column |  Type   | Collation | Nullable | Default | Storage  | Stats target | Description 
--------+---------+-----------+----------+---------+----------+--------------+-------------
 id     | integer |           |          |         | plain    |              | 
 sno    | integer |           |          |         | plain    |              | 
 name   | text    |           |          |         | extended |              | 
Indexes:
    "test_idx" btree (id, sno)

test=# select count(*) from test;
  count   
----------
 10000000
(1 row)

-- 优化前
test=# explain (analyze,verbose,costs,buffers,timing)  select distinct(id) from test;
                                                           QUERY PLAN                                                           
--------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=188693.39..188693.50 rows=11 width=4) (actual time=3554.553..3554.554 rows=11 loops=1)
   Output: id
   Group Key: test.id
   Buffers: shared hit=257 read=63438
   I/O Timings: read=120.436
   ->  Seq Scan on public.test  (cost=0.00..163693.71 rows=9999871 width=4) (actual time=0.011..1102.287 rows=10000000 loops=1)
         Output: id, sno, name
         Buffers: shared hit=257 read=63438
         I/O Timings: read=120.436
 Planning Time: 0.080 ms
 Execution Time: 3554.581 ms
(11 rows)

-- 优化后
test=# explain (analyze,verbose,costs,buffers,timing) 
with recursive t as(
 (
  select min(id) as id from test where id is not null
 ) 
 union all
 (
  select (select min(id) as id from test where test.id>t.id and test.id is not null) from t where t.id is not null 
      -- 这里的where t.id is not null 一定要加,否则就死循环了
 )
)
select distinct(id) from t;
                                                                                QUERY PLAN                                                                                 
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=61.99..63.00 rows=101 width=4) (actual time=21.264..21.267 rows=12 loops=1)
   Output: t.id
   Group Key: t.id
   Buffers: shared hit=21 read=26
   I/O Timings: read=20.847
   CTE t
     ->  Recursive Union  (cost=0.49..59.72 rows=101 width=4) (actual time=2.661..21.239 rows=12 loops=1)
           Buffers: shared hit=21 read=26
           I/O Timings: read=20.847
           ->  Result  (cost=0.49..0.50 rows=1 width=4) (actual time=2.659..2.660 rows=1 loops=1)
                 Output: $1
                 Buffers: shared read=4
                 I/O Timings: read=2.599
                 InitPlan 3 (returns $1)
                   ->  Limit  (cost=0.43..0.49 rows=1 width=4) (actual time=2.653..2.654 rows=1 loops=1)
                         Output: test_1.id
                         Buffers: shared read=4
                         I/O Timings: read=2.599
                         ->  Index Only Scan using test_idx on public.test test_1  (cost=0.43..538398.51 rows=9999871 width=4) (actual time=2.651..2.651 rows=1 loops=1)
                               Output: test_1.id
                               Index Cond: (test_1.id IS NOT NULL)
                               Heap Fetches: 1
                               Buffers: shared read=4
                               I/O Timings: read=2.599
           ->  WorkTable Scan on t t_1  (cost=0.00..5.72 rows=10 width=4) (actual time=1.546..1.546 rows=1 loops=12)
                 Output: (SubPlan 2)
                 Filter: (t_1.id IS NOT NULL)
                 Rows Removed by Filter: 0
                 Buffers: shared hit=21 read=22
                 I/O Timings: read=18.249
                 SubPlan 2
                   ->  Result  (cost=0.54..0.55 rows=1 width=4) (actual time=1.684..1.684 rows=1 loops=11)
                         Output: $3
                         Buffers: shared hit=21 read=22
                         I/O Timings: read=18.249
                         InitPlan 1 (returns $3)
                           ->  Limit  (cost=0.43..0.54 rows=1 width=4) (actual time=1.682..1.682 rows=1 loops=11)
                                 Output: test.id
                                 Buffers: shared hit=21 read=22
                                 I/O Timings: read=18.249
                                 ->  Index Only Scan using test_idx on public.test  (cost=0.43..356705.31 rows=3333290 width=4) (actual time=1.681..1.681 rows=1 loops=11)
                                       Output: test.id
                                       Index Cond: ((test.id > t_1.id) AND (test.id IS NOT NULL))
                                       Heap Fetches: 10
                                       Buffers: shared hit=21 read=22
                                       I/O Timings: read=18.249
   ->  CTE Scan on t  (cost=0.00..2.02 rows=101 width=4) (actual time=2.663..21.250 rows=12 loops=1)
         Output: t.id
         Buffers: shared hit=21 read=26
         I/O Timings: read=20.847
 Planning Time: 0.335 ms
 Execution Time: 21.316 ms
(52 rows)

-- 适用场景
	1、列上有索引
	2、唯一值的个数比较少(递归次数少)
    test=# select distinct(id) from test;
     id 
    ----
      8
      7
     10
      9
      1
      5
      2
      4
      6
      0
      3
    (11 rows)
    
 -- 使用CTE递归查询注意事项
 
 with recursive t as(
 (
  select min(id) as id from test where id is not null
 ) 
 union all
 (
  select (select min(id) as id from test where test.id>t.id and test.id is not null) from t where t.id is not null 
      -- 这里的where t.id is not null 一定要加,否则就死循环了
 )
)
 test-# select distinct(id) from t;
 id 
----
  8
   
  4
  0
 10
  6
  2
  9
  7
  3
  1
  5
(12 rows)
可以看到出现一条空数据，这是因为 select (select min(id) as id from test where test.id>t.id and test.id is not null) from t where t.id is not null;这条语句在最后一次递归的时候得到的空数据。
```

### 二、求差值

```sql
-- 场景描述
   一张小表存储着id信息，一张大表记录id活动的日志(可能只有部分id出现在这张表中)
   现在需要求出不在大表中的id
-- 测试环境
 test=# create table a(id int priamry key,info text);
CREATE TABLE
test=# create table b(id int primary key, a_id int,dt timestamp);
CREATE TABLE
test=# insert into a select a.i,'cdddddddw' from generate_series(1,1000) as a(i);
INSERT 0 1000
test=# insert into b select a.i,random()*500,now() from generate_series(1,30000000) as a(i);
INSERT 0 30000000
test=# create index b_idx on b using btree(a_id);
CREATE INDEX

-- 查询SQL
test=# select a.id from a where a.id not in (select a_id from b)；//巨慢

//not in优化(not in === <> all)
test=# explain (analyze,verbose,costs,buffers,timing) select a.id from a where a.id<>all(array(select a_id from b));
                                                           QUERY PLAN                                                           
--------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on public.a  (cost=462154.60..462183.10 rows=990 width=4) (actual time=6616.956..86580.916 rows=500 loops=1)
   Output: a.id
   Filter: (a.id <> ALL ($0))
   Rows Removed by Filter: 500
   Buffers: shared hit=128062 read=34107
   I/O Timings: read=85.522
   InitPlan 1 (returns $0)
     ->  Seq Scan on public.b  (cost=0.00..462154.60 rows=29999160 width=4) (actual time=0.022..3886.403 rows=30000000 loops=1)
           Output: b.a_id
           Buffers: shared hit=128059 read=34104
           I/O Timings: read=83.385
 Planning Time: 0.165 ms
 Execution Time: 86589.146 ms
(13 rows)


test=# explain (analyze,verbose,costs,buffers,timing) select a.id from a left join b on a.id=b.a_id where b.* is null;
                                                          QUERY PLAN                                                           
-------------------------------------------------------------------------------------------------------------------------------
 Hash Right Join  (cost=28.50..541264.63 rows=149996 width=4) (actual time=15663.944..15664.011 rows=500 loops=1)
   Output: a.id
   Inner Unique: true
   Hash Cond: (b.a_id = a.id)
   Filter: (b.* IS NULL)
   Rows Removed by Filter: 29970106
   Buffers: shared hit=127521 read=34648
   I/O Timings: read=83.050
   ->  Seq Scan on public.b  (cost=0.00..462154.60 rows=29999160 width=44) (actual time=0.014..7681.103 rows=30000000 loops=1)
         Output: b.a_id, b.*
         Buffers: shared hit=127515 read=34648
         I/O Timings: read=83.050
   -- 发现执行计划用的hash连接，将小表加载到缓冲区
   ->  Hash  (cost=16.00..16.00 rows=1000 width=4) (actual time=0.394..0.394 rows=1000 loops=1)
         Output: a.id
         Buckets: 1024  Batches: 1  Memory Usage: 44kB
         Buffers: shared hit=6
         ->  Seq Scan on public.a  (cost=0.00..16.00 rows=1000 width=4) (actual time=0.007..0.166 rows=1000 loops=1)
               Output: a.id
               Buffers: shared hit=6
 Planning Time: 0.345 ms
 Execution Time: 15664.082 ms
(21 rows)

-- 强制走索引发现效果并不好
test=# set enable_hashjoin=off; 
SET
test=# explain (analyze,verbose,costs,buffers,timing) select a.id from a left join b on a.id=b.a_id where b.* is null;
                                                                  QUERY PLAN                                                                   
-----------------------------------------------------------------------------------------------------------------------------------------------
 Merge Left Join  (cost=2353.19..1802670.16 rows=149996 width=4) (actual time=56121.336..56121.453 rows=500 loops=1)
   Output: a.id
   Merge Cond: (a.id = b.a_id)
   Filter: (b.* IS NULL)
   Rows Removed by Filter: 29970106
   Buffers: shared hit=19330076 read=5850737
   I/O Timings: read=16042.995
   ->  Index Only Scan using a_pkey on public.a  (cost=0.28..35.27 rows=1000 width=4) (actual time=0.025..1.422 rows=1000 loops=1)
         Output: a.id
         Heap Fetches: 0
         Buffers: shared hit=3 read=2
         I/O Timings: read=0.515
   -- 尽管走了索引但是实际上并没有什么作用，因为还是需要扫描全表扫描
   ->  Index Scan using b_idx on public.b  (cost=0.56..1427642.88 rows=29999160 width=44) (actual time=0.020..50818.847 rows=30000000 loops=1)
         Output: b.a_id, b.*
         Buffers: shared hit=19330073 read=5850735
         I/O Timings: read=16042.480
 Planning Time: 6.129 ms
 Execution Time: 56123.341 ms
(18 rows)

-- 优化后
test=# explain (analyze,verbose,costs,buffers,timing) with recursive t as
(
    (select min(a_id) as id from b where b.a_id is not null)
    union all
    (select (select min(a_id) as id from b where b.a_id>t.id and b.a_id is not null) from t where t.id is not null)
)
select id from a where id not in (select id from t where id is not null);
--注意事项
select id from t where id is not null；这里需要过滤由CTE产生的空数据，否则使用not in逻辑的话每次判断都是false(a中的数据和null比较是否相等会产生null)，所以不会返回数据。
                                                                              QUERY PLAN                                                                              
----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on public.a  (cost=67.24..85.74 rows=500 width=4) (actual time=10.870..11.030 rows=500 loops=1)
   Output: a.id
   Filter: (NOT (hashed SubPlan 5))
   Rows Removed by Filter: 500
   Buffers: shared hit=2017
   CTE t
     ->  Recursive Union  (cost=0.59..64.97 rows=101 width=4) (actual time=0.042..10.058 rows=502 loops=1)
           Buffers: shared hit=2011
           ->  Result  (cost=0.59..0.60 rows=1 width=4) (actual time=0.041..0.041 rows=1 loops=1)
                 Output: $1
                 Buffers: shared hit=5
                 InitPlan 3 (returns $1)
                   ->  Limit  (cost=0.56..0.59 rows=1 width=4) (actual time=0.037..0.037 rows=1 loops=1)
                         Output: b_1.a_id
                         Buffers: shared hit=5
                         ->  Index Only Scan using b_idx on public.b b_1  (cost=0.56..854029.86 rows=29999160 width=4) (actual time=0.036..0.036 rows=1 loops=1)
                               Output: b_1.a_id
                               Index Cond: (b_1.a_id IS NOT NULL)
                               Heap Fetches: 0
                               Buffers: shared hit=5
           ->  WorkTable Scan on t  (cost=0.00..6.23 rows=10 width=4) (actual time=0.019..0.019 rows=1 loops=502)
                 Output: (SubPlan 2)
                 Filter: (t.id IS NOT NULL)
                 Rows Removed by Filter: 0
                 Buffers: shared hit=2006
                 SubPlan 2
                   ->  Result  (cost=0.59..0.60 rows=1 width=4) (actual time=0.018..0.018 rows=1 loops=501)
                         Output: $3
                         Buffers: shared hit=2006
                         InitPlan 1 (returns $3)
                           ->  Limit  (cost=0.56..0.59 rows=1 width=4) (actual time=0.017..0.017 rows=1 loops=501)
                                 Output: b.a_id
                                 Buffers: shared hit=2006
                                 ->  Index Only Scan using b_idx on public.b  (cost=0.56..309678.96 rows=9999720 width=4) (actual time=0.017..0.017 rows=1 loops=501)
                                       Output: b.a_id
                                       Index Cond: ((b.a_id > t.id) AND (b.a_id IS NOT NULL))
                                       Heap Fetches: 0
                                       Buffers: shared hit=2006
   SubPlan 5
     ->  CTE Scan on t t_1  (cost=0.00..2.02 rows=100 width=4) (actual time=0.045..10.364 rows=501 loops=1)
           Output: t_1.id
           Filter: (t_1.id IS NOT NULL)
           Rows Removed by Filter: 1
           Buffers: shared hit=2011
 Planning Time: 0.361 ms
 Execution Time: 11.158 ms
(46 rows)        

-- 优化not in发现效果不明显
test=# explain (analyze,verbose,costs,buffers,timing) with recursive t as
(
    (select min(a_id) as id from b where b.a_id is not null)
    union all
    (select (select min(a_id) as id from b where b.a_id>t.id and b.a_id is not null) from t where t.id is not null)
)
select id from a where id<>any(array(select id from t where id is not null));
                                                                              QUERY PLAN                                                                              
----------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on public.a  (cost=66.99..95.49 rows=1000 width=4) (actual time=8.564..8.779 rows=1000 loops=1)
   Output: a.id
   Filter: (a.id <> ANY ($5))
   Buffers: shared hit=2017
   CTE t
     ->  Recursive Union  (cost=0.59..64.97 rows=101 width=4) (actual time=0.026..8.273 rows=502 loops=1)
           Buffers: shared hit=2011
           ->  Result  (cost=0.59..0.60 rows=1 width=4) (actual time=0.026..0.026 rows=1 loops=1)
                 Output: $1
                 Buffers: shared hit=5
                 InitPlan 3 (returns $1)
                   ->  Limit  (cost=0.56..0.59 rows=1 width=4) (actual time=0.024..0.024 rows=1 loops=1)
                         Output: b_1.a_id
                         Buffers: shared hit=5
                         ->  Index Only Scan using b_idx on public.b b_1  (cost=0.56..854029.86 rows=29999160 width=4) (actual time=0.023..0.023 rows=1 loops=1)
                               Output: b_1.a_id
                               Index Cond: (b_1.a_id IS NOT NULL)
                               Heap Fetches: 0
                               Buffers: shared hit=5
           ->  WorkTable Scan on t  (cost=0.00..6.23 rows=10 width=4) (actual time=0.016..0.016 rows=1 loops=502)
                 Output: (SubPlan 2)
                 Filter: (t.id IS NOT NULL)
                 Rows Removed by Filter: 0
                 Buffers: shared hit=2006
                 SubPlan 2
                   ->  Result  (cost=0.59..0.60 rows=1 width=4) (actual time=0.015..0.015 rows=1 loops=501)
                         Output: $3
                         Buffers: shared hit=2006
                         InitPlan 1 (returns $3)
                           ->  Limit  (cost=0.56..0.59 rows=1 width=4) (actual time=0.014..0.014 rows=1 loops=501)
                                 Output: b.a_id
                                 Buffers: shared hit=2006
                                 ->  Index Only Scan using b_idx on public.b  (cost=0.56..309678.96 rows=9999720 width=4) (actual time=0.014..0.014 rows=1 loops=501)
                                       Output: b.a_id
                                       Index Cond: ((b.a_id > t.id) AND (b.a_id IS NOT NULL))
                                       Heap Fetches: 0
                                       Buffers: shared hit=2006
   InitPlan 5 (returns $5)
     ->  CTE Scan on t t_1  (cost=0.00..2.02 rows=100 width=4) (actual time=0.028..8.498 rows=501 loops=1)
           Output: t_1.id
           Filter: (t_1.id IS NOT NULL)
           Rows Removed by Filter: 1
           Buffers: shared hit=2011
 Planning Time: 0.217 ms
 Execution Time: 8.918 ms
(45 rows)
```

### 三、求时序数据的最新值

```sql
-- 场景描述:假设有很多不同传感器不断的上传数据，我们需要查询每个传感器最新的数据

--测试环境
test=# create table sensor(id serial primary key,c_id int,c_info text);
CREATE TABLE
test=# insert into sensor(c_id,c_info) select random()*100000,'cbcdcwn' || now()::text from generate_series(1,10000000);
INSERT 0 10000000
test=# vacuum analyze;
VACUUM

-- 查询
-- 常规分组聚合(扫描sensor表时没有用并行扫描)
test=# explain (analyze,verbose,costs,buffers,timing) select id,c_id,c_info from (select id,c_id,c_info,row_number() over(partition by c_id order by id desc) as row_id from sensor) as t where row_id=1;
                                                                          QUERY PLAN                                                                           
---------------------------------------------------------------------------------------------------------------------------------------------------------------
 Subquery Scan on t  (cost=10001971370.63..10002296370.83 rows=50000 width=45) (actual time=20477.962..39974.079 rows=100001 loops=1)
   Output: t.id, t.c_id, t.c_info
   Filter: (t.row_id = 1)
   Rows Removed by Filter: 9899999
   Buffers: shared hit=93458, temp read=203017 written=203275
   ->  WindowAgg  (cost=10001971370.63..10002171370.75 rows=10000006 width=53) (actual time=20477.958..37585.449 rows=10000000 loops=1)
         Output: sensor.id, sensor.c_id, sensor.c_info, row_number() OVER (?)
         Buffers: shared hit=93458, temp read=203017 written=203275
         ->  Sort  (cost=10001971370.63..10001996370.65 rows=10000006 width=45) (actual time=20477.935..24829.575 rows=10000000 loops=1)
               Output: sensor.id, sensor.c_id, sensor.c_info
               Sort Key: sensor.c_id, sensor.id DESC
               Sort Method: external merge  Disk: 538232kB
               Buffers: shared hit=93458, temp read=203017 written=203275
               ->  Seq Scan on public.sensor  (cost=10000000000.00..10000193458.06 rows=10000006 width=45) (actual time=0.009..1582.810 rows=10000000 loops=1)
                     Output: sensor.id, sensor.c_id, sensor.c_info
                     Buffers: shared hit=93458
 Planning Time: 0.397 ms
 Execution Time: 40096.202 ms
(18 rows)


-- 索引优化(快了一点,减少分组排序成本)
test=# create index sensor_idx on sensor using btree(c_id,id desc);
CREATE INDEX

test=# explain (analyze,verbose,costs,buffers,timing) select id,c_id,c_info from (select id,c_id,c_info,row_number() over(partition by c_id order by id desc) as row_id from sensor) as t where row_id=1;
                                                                          QUERY PLAN                                                                          
--------------------------------------------------------------------------------------------------------------------------------------------------------------
 Subquery Scan on t  (cost=0.43..933520.43 rows=50000 width=45) (actual time=0.099..33806.616 rows=100001 loops=1)
   Output: t.id, t.c_id, t.c_info
   Filter: (t.row_id = 1)
   Rows Removed by Filter: 9899999
   Buffers: shared hit=9971736 read=50342
   I/O Timings: read=448.147
   ->  WindowAgg  (cost=0.43..808520.43 rows=10000000 width=53) (actual time=0.094..31321.536 rows=10000000 loops=1)
         Output: sensor.id, sensor.c_id, sensor.c_info, row_number() OVER (?)
         Buffers: shared hit=9971736 read=50342
         I/O Timings: read=448.147
         ->  Index Scan using sensor_idx on public.sensor  (cost=0.43..633520.43 rows=10000000 width=45) (actual time=0.058..17442.690 rows=10000000 loops=1)
               Output: sensor.id, sensor.c_id, sensor.c_info
               Buffers: shared hit=9971736 read=50342
               I/O Timings: read=448.147
 Planning Time: 0.492 ms
 Execution Time: 33832.267 ms
(16 rows)



-- 子查询优化(在sensor表上先做并行的顺序扫描，最后内表分组排序，没有用索引，这部分还可以优化)
test=# explain (analyze,verbose,costs,buffers,timing) select * from sensor where id in (select max(id) from sensor group by(c_id));
                                                                                   QUERY PLAN                                                                                   
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Nested Loop  (cost=766738.40..768163.88 rows=4999994 width=45) (actual time=4697.936..4970.234 rows=100001 loops=1)
   Output: sensor.id, sensor.c_id, sensor.c_info
   Inner Unique: true
   Buffers: shared hit=432617, temp read=15699 written=15761
   ->  HashAggregate  (cost=766737.97..766739.97 rows=200 width=4) (actual time=4697.906..4724.805 rows=100001 loops=1)
         Output: (max(sensor_1.id))
         Group Key: max(sensor_1.id)
         Buffers: shared hit=32341, temp read=15699 written=15761
         ->  Finalize GroupAggregate  (cost=708197.60..765490.00 rows=99837 width=8) (actual time=3040.107..4655.762 rows=100001 loops=1)
               Output: max(sensor_1.id), sensor_1.c_id
               Group Key: sensor_1.c_id
               Buffers: shared hit=32341, temp read=15699 written=15761
               ->  Gather Merge  (cost=708197.60..763493.26 rows=199674 width=8) (actual time=3040.094..4569.706 rows=300003 loops=1)
                     Output: sensor_1.c_id, (PARTIAL max(sensor_1.id))
                     Workers Planned: 2
                     Workers Launched: 2
                     Buffers: shared hit=93512, temp read=45397 written=45576
                     ->  Partial GroupAggregate  (cost=707197.58..739445.91 rows=99837 width=8) (actual time=2993.005..4493.487 rows=100001 loops=3)
                           Output: sensor_1.c_id, PARTIAL max(sensor_1.id)
                           Group Key: sensor_1.c_id
                           Buffers: shared hit=93512, temp read=45397 written=45576
                           Worker 0: actual time=2945.448..4493.257 rows=100001 loops=1
                             Buffers: shared hit=31540, temp read=15258 written=15318
                           Worker 1: actual time=2999.663..4494.931 rows=100001 loops=1
                             Buffers: shared hit=29631, temp read=14440 written=14497
                           ->  Sort  (cost=707197.58..717614.23 rows=4166661 width=8) (actual time=2992.984..3645.708 rows=3333333 loops=3)
                                 Output: sensor_1.c_id, sensor_1.id
                                 Sort Key: sensor_1.c_id
                                 Sort Method: external merge  Disk: 60992kB
                                 Worker 0:  Sort Method: external merge  Disk: 59424kB
                                 Worker 1:  Sort Method: external merge  Disk: 55808kB
                                 Buffers: shared hit=93512, temp read=45397 written=45576
                                 Worker 0: actual time=2945.421..3622.255 rows=3371891 loops=1
                                   Buffers: shared hit=31540, temp read=15258 written=15318
                                 Worker 1: actual time=2999.637..3663.176 rows=3167628 loops=1
                                   Buffers: shared hit=29631, temp read=14440 written=14497
                                 ->  Parallel Seq Scan on public.sensor sensor_1  (cost=0.00..135124.61 rows=4166661 width=8) (actual time=0.390..644.515 rows=3333333 loops=3)
                                       Output: sensor_1.c_id, sensor_1.id
                                       Buffers: shared hit=93458
                                       Worker 0: actual time=0.013..687.599 rows=3371891 loops=1
                                         Buffers: shared hit=31513
                                       Worker 1: actual time=0.013..644.307 rows=3167628 loops=1
                                         Buffers: shared hit=29604
   ->  Index Scan using sensor_pkey on public.sensor  (cost=0.43..8.45 rows=1 width=45) (actual time=0.002..0.002 rows=1 loops=100001)
         Output: sensor.id, sensor.c_id, sensor.c_info
         Index Cond: (sensor.id = (max(sensor_1.id)))
         Buffers: shared hit=400276
 Planning Time: 0.279 ms
 Execution Time: 4990.671 ms
(49 rows)


-- 优化in,发现优化程度不是特别大(仅优化了Nestloop--索引扫描)
test=# explain (analyze,verbose,costs,buffers,timing) select * from sensor where id=any(array((select max(id) from sensor group by(c_id))));
                                                                                 QUERY PLAN                                                                                 
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Index Scan using sensor_pkey on public.sensor  (cost=765490.44..765538.53 rows=10 width=45) (actual time=4773.750..4902.967 rows=100001 loops=1)
   Output: sensor.id, sensor.c_id, sensor.c_info
   Index Cond: (sensor.id = ANY ($1))
   Buffers: shared hit=335824, temp read=14740 written=14798
   InitPlan 1 (returns $1)
     ->  Finalize GroupAggregate  (cost=708197.60..765490.00 rows=99837 width=8) (actual time=3113.261..4722.146 rows=100001 loops=1)
           Output: max(sensor_1.id), sensor_1.c_id
           Group Key: sensor_1.c_id
           Buffers: shared hit=30359, temp read=14740 written=14798
           ->  Gather Merge  (cost=708197.60..763493.26 rows=199674 width=8) (actual time=3113.243..4627.487 rows=300003 loops=1)
                 Output: sensor_1.c_id, (PARTIAL max(sensor_1.id))
                 Workers Planned: 2
                 Workers Launched: 2
                 Buffers: shared hit=93512, temp read=45273 written=45451
                 ->  Partial GroupAggregate  (cost=707197.58..739445.91 rows=99837 width=8) (actual time=3101.076..4594.653 rows=100001 loops=3)
                       Output: sensor_1.c_id, PARTIAL max(sensor_1.id)
                       Group Key: sensor_1.c_id
                       Buffers: shared hit=93512, temp read=45273 written=45451
                       Worker 0: actual time=3095.630..4630.689 rows=100001 loops=1
                         Buffers: shared hit=32698, temp read=15777 written=15839
                       Worker 1: actual time=3094.667..4602.505 rows=100001 loops=1
                         Buffers: shared hit=30455, temp read=14756 written=14814
                       ->  Sort  (cost=707197.58..717614.23 rows=4166661 width=8) (actual time=3101.052..3747.218 rows=3333333 loops=3)
                             Output: sensor_1.c_id, sensor_1.id
                             Sort Key: sensor_1.c_id
                             Sort Method: external merge  Disk: 57232kB
                             Worker 0:  Sort Method: external merge  Disk: 61616kB
                             Worker 1:  Sort Method: external merge  Disk: 57360kB
                             Buffers: shared hit=93512, temp read=45273 written=45451
                             Worker 0: actual time=3095.605..3753.718 rows=3495791 loops=1
                               Buffers: shared hit=32698, temp read=15777 written=15839
                             Worker 1: actual time=3094.638..3754.368 rows=3255796 loops=1
                               Buffers: shared hit=30455, temp read=14756 written=14814
                             ->  Parallel Seq Scan on public.sensor sensor_1  (cost=0.00..135124.61 rows=4166661 width=8) (actual time=0.012..678.574 rows=3333333 loops=3)
                                   Output: sensor_1.c_id, sensor_1.id
                                   Buffers: shared hit=93458
                                   Worker 0: actual time=0.013..709.080 rows=3495791 loops=1
                                     Buffers: shared hit=32671
                                   Worker 1: actual time=0.013..697.339 rows=3255796 loops=1
                                     Buffers: shared hit=30428
 Planning Time: 0.202 ms
 Execution Time: 4915.497 ms
(42 rows)

-- 索引优化(效果也不好,优化了排序max(id),并行索引扫描,效果还不错)
test=# create index sensor_idx on sensor using btree(c_id,id desc);
CREATE INDEX
test=# explain (analyze,verbose,costs,buffers,timing) select * from sensor where id=any(array((select max(id) from sensor group by(c_id))));
                                                                                          QUERY PLAN                                                                                        
  
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
--
 Index Scan using sensor_pkey on public.sensor  (cost=249634.72..249682.81 rows=10 width=45) (actual time=3309.410..3756.268 rows=100001 loops=1)
   Output: sensor.id, sensor.c_id, sensor.c_info
   Index Cond: (sensor.id = ANY ($1))
   Buffers: shared hit=365175 read=1770
   I/O Timings: read=152.710
   InitPlan 1 (returns $1)
     ->  Finalize GroupAggregate  (cost=1000.46..249634.29 rows=101384 width=8) (actual time=31.752..3218.228 rows=100001 loops=1)
           Output: max(sensor_1.id), sensor_1.c_id
           Group Key: sensor_1.c_id
           Buffers: shared hit=61480
           ->  Gather Merge  (cost=1000.46..247606.61 rows=202768 width=8) (actual time=31.653..3090.929 rows=125358 loops=1)
                 Output: sensor_1.c_id, (PARTIAL max(sensor_1.id))
                 Workers Planned: 2
                 Workers Launched: 2
                 Buffers: shared hit=345080
                 ->  Partial GroupAggregate  (cost=0.43..223202.13 rows=101384 width=8) (actual time=0.155..2432.858 rows=41786 loops=3)
                       Output: sensor_1.c_id, PARTIAL max(sensor_1.id)
                       Group Key: sensor_1.c_id
                       Buffers: shared hit=345080
                       Worker 0: actual time=0.140..2833.062 rows=50847 loops=1
                         Buffers: shared hit=140340
                       Worker 1: actual time=0.198..2834.833 rows=51850 loops=1
                         Buffers: shared hit=143260
                       ->  Parallel Index Only Scan using sensor_idx on public.sensor sensor_1  (cost=0.43..201354.98 rows=4166661 width=8) (actual time=0.084..993.822 rows=3333333 loops=3
)
                             Output: sensor_1.c_id, sensor_1.id
                             Heap Fetches: 0
                             Buffers: shared hit=345080
                             Worker 0: actual time=0.088..1141.567 rows=4062600 loops=1
                               Buffers: shared hit=140340
                             Worker 1: actual time=0.101..1146.979 rows=4154466 loops=1
                               Buffers: shared hit=143260
 Planning Time: 1.825 ms
 Execution Time: 3768.031 ms
(33 rows)





-- 基本想法:递归的取出每个传感器的最新的值，但是出现个问题，子查询中只能出现一个列
-- 解决办法：利用自定义类型
-- 可以看到用CTE的效果和上面第二个方法差不多的效果(在没有查询的情况下，CTE循环可能会非常差,非常消耗CPU)
-- 这里为啥CTE效果不好，因为这个查询不管怎样都不可避免的需要全索引扫描和第二个方法的效果差不多，没有像前面的CTE那样利用CTE避免了大量的索引页的扫描。
test=# with recursive t as
(
 (select c_id,c_info from sensor where id is not null order by c_id,id limit 1)
 union all
 (select (select c_id,c_info from sensor where sensor.id is not null and sensor.id>t.c_id) from t where t.c_id is not null)
)
select * from t;
ERROR:  subquery must return only one column
LINE 5:  (select (select c_id,c_info from sensor where sensor.id is ...
                  
         
-- 定义自定义类型
test=# create type r as (c_id int,c_info text);
CREATE TYPE
                
test=# create index sensor_idx on sensor using btree(c_id,id desc);
CREATE INDEX
                  
test=# explain (analyze,verbose,costs,buffers,timing) with recursive t as (    
  (    
    select (c_id,c_info)::r as r from sensor where id in (select id from sensor where c_id is not null order by c_id,id desc limit 1)   
  )    
  union all    
  (    
    select (  
      select (c_id,c_info)::r as r from sensor where id in (select id from sensor tt where tt.c_id>(t.r).c_id and tt.c_id is not null order by c_id,id desc limit 1)   
    ) from t where (t.r).c_id is not null  
  )      
)     
select (t.r).c_id, (t.r).c_info from t where t.* is not null;
                                                                                        QUERY PLAN                                                                                        
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 CTE Scan on t  (cost=907.06..909.08 rows=100 width=36) (actual time=0.210..3740.646 rows=100001 loops=1)
   Output: (t.r).c_id, (t.r).c_info
   Filter: (t.* IS NOT NULL)
   Rows Removed by Filter: 1
   Buffers: shared hit=700552, temp written=878
   CTE t
     ->  Recursive Union  (cost=0.91..907.06 rows=101 width=32) (actual time=0.197..3517.265 rows=100002 loops=1)
           Buffers: shared hit=700552
           ->  Nested Loop  (cost=0.91..8.94 rows=1 width=32) (actual time=0.196..0.197 rows=1 loops=1)
                 Output: ROW(sensor_1.c_id, sensor_1.c_info)::r
                 Inner Unique: true
                 Buffers: shared hit=8
                 ->  HashAggregate  (cost=0.48..0.49 rows=1 width=4) (actual time=0.175..0.175 rows=1 loops=1)
                       Output: sensor_2.id
                       Group Key: sensor_2.id
                       Buffers: shared hit=4
                       ->  Limit  (cost=0.43..0.46 rows=1 width=8) (actual time=0.167..0.168 rows=1 loops=1)
                             Output: sensor_2.id, sensor_2.c_id
                             Buffers: shared hit=4
                             ->  Index Only Scan using sensor_idx on public.sensor sensor_2  (cost=0.43..284688.54 rows=10000006 width=8) (actual time=0.166..0.166 rows=1 loops=1)
                                   Output: sensor_2.id, sensor_2.c_id
                                   Index Cond: (sensor_2.c_id IS NOT NULL)
                                   Heap Fetches: 0
                                   Buffers: shared hit=4
                 ->  Index Scan using sensor_pkey on public.sensor sensor_1  (cost=0.43..8.45 rows=1 width=45) (actual time=0.015..0.015 rows=1 loops=1)
                       Output: sensor_1.id, sensor_1.c_id, sensor_1.c_info
                       Index Cond: (sensor_1.id = sensor_2.id)
                       Buffers: shared hit=4
           ->  WorkTable Scan on t t_1  (cost=0.00..89.61 rows=10 width=32) (actual time=0.033..0.034 rows=1 loops=100002)
                 Output: (SubPlan 1)
                 Filter: ((t_1.r).c_id IS NOT NULL)
                 Rows Removed by Filter: 0
                 Buffers: shared hit=700544
                 SubPlan 1
                   ->  Nested Loop  (cost=0.91..8.94 rows=1 width=32) (actual time=0.031..0.031 rows=1 loops=100001)
                         Output: ROW(sensor.c_id, sensor.c_info)::r
                         Inner Unique: true
                         Buffers: shared hit=700544
                         ->  HashAggregate  (cost=0.48..0.49 rows=1 width=4) (actual time=0.023..0.023 rows=1 loops=100001)
                               Output: tt.id
                               Group Key: tt.id
                               Buffers: shared hit=300272
                               ->  Limit  (cost=0.43..0.47 rows=1 width=8) (actual time=0.021..0.021 rows=1 loops=100001)
                                     Output: tt.id, tt.c_id
                                     Buffers: shared hit=300272
                                     ->  Index Only Scan using sensor_idx on public.sensor tt  (cost=0.43..103231.14 rows=3333335 width=8) (actual time=0.020..0.020 rows=1 loops=100001)
                                           Output: tt.id, tt.c_id
                                           Index Cond: ((tt.c_id > (t_1.r).c_id) AND (tt.c_id IS NOT NULL))
                                           Heap Fetches: 0
                                           Buffers: shared hit=300272
                         ->  Index Scan using sensor_pkey on public.sensor  (cost=0.43..8.45 rows=1 width=45) (actual time=0.006..0.006 rows=1 loops=100000)
                               Output: sensor.id, sensor.c_id, sensor.c_info
                               Index Cond: (sensor.id = tt.id)
                               Buffers: shared hit=400272
 Planning Time: 0.943 ms
 Execution Time: 3763.342 ms
(56 rows)
```

