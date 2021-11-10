---
tags: [Database]
title: '[Database] Horizontal partition'
created: '2021-11-10T03:14:48.714Z'
modified: '2021-11-10T04:55:19.853Z'
---

# [Database] Horizontal partition

# Create grades table
```sql
create table grades (
id serial primary key, 
 g int,
 name text 
); 

```
init data
```sql
insert into grades (g,
name  ) 
select 
random()*100,
substring(md5(random()::text ),0,floor(random()*31)::int)
 from generate_series(0, 50000000);
```
the value of g here ranges from 0 to 100;

clean dead rows if needed
```sql
vacuum (analyze, verbose, full);
```

# Create table
```sql
create table grades_parts (
id serial not null,
g int not null
) partition by range(g);

```

# Create 4 partitions table
g value ranges from 00 to 34, not includes 35:
```sql
create table g0035 (
like grades_parts including indexes
);
```
g value ranges from 35 to 59:
```sql
create table g3560 (
like grades_parts including indexes
);
```
g value ranges from 60 to 79:
```sql
create table g6080 (
like grades_parts including indexes
);
```
g value ranges from 80 t0 99:
```sql
create table g80100 (
like grades_parts including indexes
);
```

# Alter table

```sql
alter table grades_parts attach partition g0035 for values from (0) to (35);
```
```sql
alter table grades_parts attach partition g3560 for values from (35) to (60);
```
```sql
alter table grades_parts attach partition g6080 for values from (60) to (80);
```
```sql
alter table grades_parts attach partition g80100 for values from (80) to (100);
```

# Insert data
since g value ranges from 00 to 99, so I have to remove rows which have g equal to 100.

```sql
insert into grades_parts select * from grades where g != 100;

```

# Test

```bash
postgres=# select max(g) from g0035;
 max 
-----
  34
(1 row)

```

```bash

postgres=# select count(*) from g0035;
 count  
--------
 345345
(1 row)

postgres=# select count(*) from g3560;
 count  
--------
 250374
(1 row)
```

# Create index

only need to create index on master, and then the dbms auto create index on child tables.

```sql
create index grade_idx on grades_parts(g);
```

```bash
postgres=# \d grades_parts;
                      Partitioned table "public.grades_parts"
 Column |  Type   | Collation | Nullable |                 Default                  
--------+---------+-----------+----------+------------------------------------------
 id     | integer |           | not null | nextval('grades_parts_id_seq'::regclass)
 g      | integer |           | not null | 
Partition key: RANGE (g)
Indexes:
    "grade_idx" btree (g)
Number of partitions: 4 (Use \d+ to list them.)
```

```bash
postgres=# \d g0035;
               Table "public.g0035"
 Column |  Type   | Collation | Nullable | Default 
--------+---------+-----------+----------+---------
 id     | integer |           | not null | 
 g      | integer |           | not null | 
Partition of: grades_parts FOR VALUES FROM (0) TO (35)
Indexes:
    "g0035_g_idx" btree (g)

```

# Query 

```bash
postgres=# explain analyze select count(*) from grades_parts where g = 30;
                                                                    QUERY PLAN                                                                     
---------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=228.90..228.91 rows=1 width=8) (actual time=10.421..10.423 rows=1 loops=1)
   ->  Index Only Scan using g0035_g_idx on g0035 grades_parts  (cost=0.42..204.84 rows=9624 width=0) (actual time=0.190..6.520 rows=9940 loops=1)
         Index Cond: (g = 30)
         Heap Fetches: 0
 Planning Time: 1.200 ms
 Execution Time: 10.609 ms
(6 rows)

```

# size of index

```bash
postgres=# select pg_relation_size(oid), relname from pg_class order by pg_relation_size(oid) desc;
 pg_relation_size |                    relname                    
------------------+-----------------------------------------------
       7809859584 | students
       1124696064 | g_idx
       1123115008 | students_pkey
         55197696 | grades
         41795584 | employees
         38789120 | g_index
         36249600 | temp
         22495232 | grades_pkey
         22487040 | employees_pkey
         12525568 | g0035
          9076736 | g3560
          7249920 | g80100
          7249920 | g6080
          2433024 | g0035_g_idx
          1777664 | g3560_g_idx
          1409024 | g80100_g_idx
          1409024 | g6080_g_idx

```
**g_index** and **grades_pkey** are indexes of **grades** table

# Enable pruning

```bash
postgres=# show ENABLE_PARTITION_PRUNING;
 enable_partition_pruning 
--------------------------
 on
(1 row)

```

`ENABLE_PARTITION_PRUNING` should be on, what happen if it's off? the whole partition thing is useless.

```bash
postgres=# set ENABLE_PARTITION_PRUNING = off;
SET
postgres=# explain select count(*) from grades_parts where g = 30;
                                                  QUERY PLAN                                                  
--------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=290.11..290.12 rows=1 width=8)
   ->  Append  (cost=0.42..266.04 rows=9627 width=0)
         ->  Index Only Scan using g0035_g_idx on g0035 grades_parts_1  (cost=0.42..204.84 rows=9624 width=0)
               Index Cond: (g = 30)
         ->  Index Only Scan using g3560_g_idx on g3560 grades_parts_2  (cost=0.42..4.44 rows=1 width=0)
               Index Cond: (g = 30)
         ->  Index Only Scan using g6080_g_idx on g6080 grades_parts_3  (cost=0.29..4.31 rows=1 width=0)
               Index Cond: (g = 30)
         ->  Index Only Scan using g80100_g_idx on g80100 grades_parts_4  (cost=0.29..4.31 rows=1 width=0)
               Index Cond: (g = 30)
(10 rows)

postgres=# set ENABLE_PARTITION_PRUNING = on;
SET
postgres=# 
postgres=# explain select count(*) from grades_parts where g = 30;
                                              QUERY PLAN                                              
------------------------------------------------------------------------------------------------------
 Aggregate  (cost=228.90..228.91 rows=1 width=8)
   ->  Index Only Scan using g0035_g_idx on g0035 grades_parts  (cost=0.42..204.84 rows=9624 width=0)
         Index Cond: (g = 30)
(3 rows)


```

# Advantages of Partitioning
1. improves query performance when accessing a single partition
2. sequential scan vs scattered index scan
3. easy bulk loading
4. archive old data that are barely accessed into cheap storage

# Disadvantages of Partitioning
1. Updating on partion key which move row from a partition to another partition is slow and fail sometime.
2. Inefficient queries could accidentally scan all partitions resulting in slow performance.
3. Schema changing could be challenging.

# Auto create partition in postgres

https://github.com/hnasr/javascript_playground/tree/master/automate_partitions

