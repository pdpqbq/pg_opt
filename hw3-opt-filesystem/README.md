# Настройка файловой системы

❖ Секционируем по дням тайские перевозки:
https://github.com/aeuge/postgres16book/tree/main/database

❖ Тестируем производительность:
- запрос уходит в 1 секцию
- запрос уходит в 10 секций
- запрос уходит во все секции
- разносим по 3 разным дискам и тестируем производительность

Задание со *: сравнить производительность COPY vs PGLOADER

### Тестовый стенд

VM 8 RAM 4 CPU

Установка и настройка ПО, загрузка БД thai_medium
```
apt update && apt upgrade -y && apt install postgresql-15 fio atop -y

su - postgres

cat >> /etc/postgresql/15/main/postgresql.conf << EOL

synchronous_commit = on
listen_addresses = '*'

# Connectivity
max_connections = 300
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = '4096 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '6 GB'
effective_io_concurrency = 200
random_page_cost = 1.25

# Monitoring
shared_preload_libraries = 'pg_stat_statements'    # per statement resource usage stats
track_io_timing=on        # measure exact block IO times
track_functions=pl        # track execution times of pl-language procedures if any

# Checkpointing: 
checkpoint_timeout  = '15 min' 
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '1024 MB'

# WAL writing
wal_compression = on
wal_buffers = -1    # auto-tuned by Postgres till maximum of segment size (16MB by default)
wal_writer_delay = 200ms
wal_writer_flush_after = 1MB

# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0

# Parallel queries: 
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_maintenance_workers = 2
max_parallel_workers = 4
parallel_leader_participation = on

# Advanced features 
enable_partitionwise_join = on 
enable_partitionwise_aggregate = on
jit = on
track_wal_io_timing = on
maintenance_io_concurrency = 200
wal_recycle = on
EOL

pg_ctlcluster 15 main stop && pg_ctlcluster 15 main start

wget https://storage.googleapis.com/thaibus/thai_medium.tar.gz && tar -xf thai_medium.tar.gz && psql < thai.sql
```

Состояние процесса загрузки БД можно наблюдать через pg_stat_progress_copy
```
postgres=# select * from pg_stat_progress_copy;
 pid | datid | datname | relid |  command  | type | bytes_processed | bytes_total | tuples_processed | tuples_excluded 
-----+-------+---------+-------+-----------+------+-----------------+-------------+------------------+-----------------
 567 | 16448 | thai    | 16494 | COPY FROM | PIPE |      1532947596 |           0 |         25488999 |               0
(1 row)
```

## Партиционирование

Партиционируем по дням таблицу ride, всего будет 1000 партиций

Создаем таблицу ride_p и партиции в ней, создаем ФК, копируем данные из ride
```
thai=# set search_path = 'book';

thai=# select min(startdate), max(startdate), count(distinct(startdate)) from ride;
    min     |    max     | count 
------------+------------+-------
 2000-01-01 | 2002-09-26 |  1000

CREATE TABLE book.ride_p (
    id SERIAL,
    startdate date,
    fkbus integer,
    fkschedule integer
) PARTITION BY LIST (startdate);

DO $do$
DECLARE
    v_date date;
BEGIN
    FOR v_date IN SELECT distinct(startdate) FROM ride ORDER BY 1 ASC
    LOOP
        EXECUTE format('DROP TABLE IF EXISTS book."ride_part_%s"', v_date);
        EXECUTE format('CREATE TABLE book."ride_part_%s" PARTITION OF ride_p FOR VALUES IN (''%s'')', v_date, v_date);
    END LOOP;
END;
$do$;

thai=# \d ride
                               Table "book.ride"
   Column   |  Type   | Collation | Nullable |             Default              
------------+---------+-----------+----------+----------------------------------
 id         | integer |           | not null | nextval('ride_id_seq'::regclass)
 startdate  | date    |           |          | 
 fkbus      | integer |           |          | 
 fkschedule | integer |           |          | 
Indexes:
    "ride_pkey" PRIMARY KEY, btree (id)
Foreign-key constraints:
    "ride_fkbus_fkey" FOREIGN KEY (fkbus) REFERENCES bus(id)
    "ride_fkschedule_fkey" FOREIGN KEY (fkschedule) REFERENCES schedule(id)
Referenced by:
    TABLE "tickets" CONSTRAINT "tickets_fkride_fkey" FOREIGN KEY (fkride) REFERENCES ride(id)

ALTER TABLE book.ride_p ADD CONSTRAINT ride_fkbus_fkey FOREIGN KEY (fkbus) REFERENCES bus(id);
ALTER TABLE book.ride_p ADD CONSTRAINT ride_fkschedule_fkey FOREIGN KEY (fkschedule) REFERENCES schedule(id);

thai=# \d ride_p
                         Partitioned table "book.ride_p"
   Column   |  Type   | Collation | Nullable |              Default               
------------+---------+-----------+----------+------------------------------------
 id         | integer |           | not null | nextval('ride_p_id_seq'::regclass)
 startdate  | date    |           |          | 
 fkbus      | integer |           |          | 
 fkschedule | integer |           |          | 
Partition key: LIST (startdate)
Foreign-key constraints:
    "ride_fkbus_fkey" FOREIGN KEY (fkbus) REFERENCES bus(id)
    "ride_fkschedule_fkey" FOREIGN KEY (fkschedule) REFERENCES schedule(id)
Number of partitions: 1000 (Use \d+ to list them.)

thai=# insert into ride_p select * from ride;
INSERT 0 1500000
```

Ограничение для tickets не удается перенести
```
thai=# ALTER TABLE ONLY book.tickets DROP CONSTRAINT tickets_fkride_fkey;
thai=# ALTER TABLE ONLY book.tickets ADD CONSTRAINT tickets_fkride_fkey FOREIGN KEY (fkride) REFERENCES ride_p(id);
ERROR:  there is no unique constraint matching given keys for referenced table "ride_p"

thai=# ALTER TABLE ride_p ADD CONSTRAINT ride_p_pkey PRIMARY KEY (id);
ERROR:  unique constraint on partitioned table must include all partitioning columns
DETAIL:  PRIMARY KEY constraint on table "ride_p" lacks column "startdate" which is part of the partition key.

thai=# ALTER TABLE ride_p ADD CONSTRAINT ride_p_pkey PRIMARY KEY (id, startdate);
ALTER TABLE

thai=# ALTER TABLE ONLY book.tickets ADD CONSTRAINT tickets_fkride_fkey FOREIGN KEY (fkride) REFERENCES ride_p(id, startdate);
ERROR:  number of referencing and referenced columns for foreign key disagree
```

Переименовываем таблицы
```
thai=# alter table ride rename to ride_no_parts;
ALTER TABLE
thai=# alter table ride_p rename to ride;
ALTER TABLE
```

Создаем скрипты для тестовой нагрузки, запрос получает данные о поездке по номеру билета
```
SELECT r.startdate, t.fio, t.contact, bf.city, bf.name, bt.city, bt.name, b.model FROM book.tickets t
JOIN book.ride r
ON t.fkride = r.id
JOIN book.schedule s
ON r.fkschedule = s.id
JOIN book.busroute br
ON s.fkroute = br.id
JOIN book.busstation bf
ON br.fkbusstationfrom = bf.id
JOIN book.busstation bt
ON br.fkbusstationto = bt.id
JOIN book.bus b
ON r.fkbus = b.id
WHERE t.id = 11223344;

 startdate  |      fio       |          contact          |   city   |     name     |  city   | name  |   model   
------------+----------------+---------------------------+----------+--------------+---------+-------+-----------
 2002-04-05 | ISAEVA MATRENA | {"phone": "+72884196136"} | Bankgkok | Suvarnabhumi | Pattaya | South | Toyota N1
```

Ищем номера билетов за весь период, одну партицию, 10 партиций и создаем скрипты с этими значениями

Позже выяснилось, что ограничение диапазона номеров билетов не влияет
```
select min(id), max(id) from tickets;

 min |   max    
-----+----------
   1 | 53997475

SELECT min(foo.id), max(foo.id) FROM (
SELECT t.id, r.startdate, t.fio, t.contact, bf.city, bf.name, bt.city, bt.name, b.model FROM book.tickets t
JOIN book.ride r
ON t.fkride = r.id
JOIN book.schedule s
ON r.fkschedule = s.id
JOIN book.busroute br
ON s.fkroute = br.id
JOIN book.busstation bf
ON br.fkbusstationfrom = bf.id
JOIN book.busstation bt
ON br.fkbusstationto = bt.id
JOIN book.bus b
ON r.fkbus = b.id
WHERE r.startdate = '2002-04-05') AS foo;

  min   |   max    
--------+----------
 422760 | 53617276

SELECT min(foo.id), max(foo.id) FROM (
SELECT t.id, r.startdate, t.fio, t.contact, bf.city, bf.name, bt.city, bt.name, b.model FROM book.tickets t
JOIN book.ride r
ON t.fkride = r.id
JOIN book.schedule s
ON r.fkschedule = s.id
JOIN book.busroute br
ON s.fkroute = br.id
JOIN book.busstation bf
ON br.fkbusstationfrom = bf.id
JOIN book.busstation bt
ON br.fkbusstationto = bt.id
JOIN book.bus b
ON r.fkbus = b.id
WHERE r.startdate BETWEEN ('2002-04-01') and ('2002-04-10')) AS foo;

  min   |   max    
--------+----------
 416069 | 53624027

cat > ~/workload_all.sql << EOL

\set r random(1, 53997475)
SELECT r.startdate, t.fio, t.contact, bf.city, bf.name, bt.city, bt.name, b.model FROM book.tickets t
JOIN book.ride r
ON t.fkride = r.id
JOIN book.schedule s
ON r.fkschedule = s.id
JOIN book.busroute br
ON s.fkroute = br.id
JOIN book.busstation bf
ON br.fkbusstationfrom = bf.id
JOIN book.busstation bt
ON br.fkbusstationto = bt.id
JOIN book.bus b
ON r.fkbus = b.id
WHERE t.id = :r;
EOL

cat > ~/workload_1.sql << EOL

\set r random(422760, 53617276)
SELECT r.startdate, t.fio, t.contact, bf.city, bf.name, bt.city, bt.name, b.model FROM book.tickets t
JOIN book.ride r
ON t.fkride = r.id
JOIN book.schedule s
ON r.fkschedule = s.id
JOIN book.busroute br
ON s.fkroute = br.id
JOIN book.busstation bf
ON br.fkbusstationfrom = bf.id
JOIN book.busstation bt
ON br.fkbusstationto = bt.id
JOIN book.bus b
ON r.fkbus = b.id
WHERE t.id = :r AND r.startdate = '2002-04-05';
EOL

cat > ~/workload_10.sql << EOL

\set r random(416069, 53624027)
SELECT r.startdate, t.fio, t.contact, bf.city, bf.name, bt.city, bt.name, b.model FROM book.tickets t
JOIN book.ride r
ON t.fkride = r.id
JOIN book.schedule s
ON r.fkschedule = s.id
JOIN book.busroute br
ON s.fkroute = br.id
JOIN book.busstation bf
ON br.fkbusstationfrom = bf.id
JOIN book.busstation bt
ON br.fkbusstationto = bt.id
JOIN book.bus b
ON r.fkbus = b.id
WHERE t.id = :r AND r.startdate BETWEEN ('2002-04-01') and ('2002-04-10');
EOL

cat > ~/workload_noparts.sql << EOL

\set r random(1, 53997475)
SELECT r.startdate, t.fio, t.contact, bf.city, bf.name, bt.city, bt.name, b.model FROM book.tickets t
JOIN book.ride_no_parts r
ON t.fkride = r.id
JOIN book.schedule s
ON r.fkschedule = s.id
JOIN book.busroute br
ON s.fkroute = br.id
JOIN book.busstation bf
ON br.fkbusstationfrom = bf.id
JOIN book.busstation bt
ON br.fkbusstationto = bt.id
JOIN book.bus b
ON r.fkbus = b.id
WHERE t.id = :r;
EOL
```

EXPLAIN для запроса всех партиций
```
->  Index Scan using "ride_part_2000-01-01_pkey" on "ride_part_2000-01-01" r_1  (cost=0.28..2.79 rows=1 width=16)
        Index Cond: (id = t.fkride)
...
->  Index Scan using "ride_part_2002-09-26_pkey" on "ride_part_2002-09-26" r_1000  (cost=0.28..2.79 rows=1 width=16)
        Index Cond: (id = t.fkride)
```

EXPLAIN для запроса 1 партиции
``` 
->  Index Scan using "ride_part_2002-04-05_pkey" on "ride_part_2002-04-05" r  (cost=0.28..2.80 rows=1 width=16)
        Index Cond: ((id = t.fkride) AND (startdate = '2002-04-05'::date))     
```

EXPLAIN для запроса 10 партиций
```
->  Index Scan using "ride_part_2002-04-01_pkey" on "ride_part_2002-04-01" r_1  (cost=0.28..2.80 rows=1 width=16)
        Index Cond: ((id = t.fkride) AND (startdate >= '2002-04-01'::date) AND (startdate <= '2002-04-10'::date))
...
->  Index Scan using "ride_part_2002-04-10_pkey" on "ride_part_2002-04-10" r_10  (cost=0.28..2.80 rows=1 width=16)
        Index Cond: ((id = t.fkride) AND (startdate >= '2002-04-01'::date) AND (startdate <= '2002-04-10'::date))
```

EXPLAIN для запроса без партиций
```
->  Index Scan using ride_pkey on ride_no_parts r  (cost=0.43..2.95 rows=1 width=16)
        Index Cond: (id = t.fkride)
```   

Тест локально
```
$ pgbench -c 10 -j 4 -T 10 -f ~/workload_noparts.sql -p 5432 -U postgres thai -n | grep -E 'transaction|tps'
transaction type: /var/lib/postgresql/workload_noparts.sql
number of transactions actually processed: 49945
number of failed transactions: 0 (0.000%)
tps = 4998.404254 (without initial connection time)

$ pgbench -c 10 -j 4 -T 10 -f ~/workload_1.sql -p 5432 -U postgres thai -n | grep -E 'transaction|tps'
transaction type: /var/lib/postgresql/workload_1.sql
number of transactions actually processed: 55708
number of failed transactions: 0 (0.000%)
tps = 5573.978282 (without initial connection time)

$ pgbench -c 10 -j 4 -T 10 -f ~/workload_10.sql -p 5432 -U postgres thai -n | grep -E 'transaction|tps'
transaction type: /var/lib/postgresql/workload_10.sql
number of transactions actually processed: 41036
number of failed transactions: 0 (0.000%)
tps = 4099.368222 (without initial connection time)

$ pgbench -c 10 -j 4 -T 10 -f ~/workload_all.sql -p 5432 -U postgres thai -n | grep -E 'transaction|tps'
transaction type: /var/lib/postgresql/workload_all.sql
number of transactions actually processed: 223
number of failed transactions: 0 (0.000%)
tps = 21.802549 (without initial connection time)
```

Тест по сети
```
$ pgbench -c 10 -j 4 -T 10 -f ~/workload_noparts.sql -U postgres -h 192.168.1.223 -p 5432 thai -n
number of transactions actually processed: 16708
number of failed transactions: 0 (0.000%)
tps = 2173.316954 (without initial connection time)

$ pgbench -c 10 -j 4 -T 10 -f ~/workload_1.sql -U postgres -h 192.168.1.223 -p 5432 thai -n
number of transactions actually processed: 23100
number of failed transactions: 0 (0.000%)
tps = 2381.682449 (without initial connection time)

$ pgbench -c 10 -j 4 -T 10 -f ~/workload_10.sql -U postgres -h 192.168.1.223 -p 5432 thai -n
number of transactions actually processed: 20494
number of failed transactions: 0 (0.000%)
tps = 2063.883925 (without initial connection time)

$ pgbench -c 10 -j 4 -T 10 -f ~/workload_all.sql -U postgres -h 192.168.1.223 -p 5432 thai -n
number of transactions actually processed: 221
number of failed transactions: 0 (0.000%)
tps = 22.094554 (without initial connection time)
```

![](img/img1.png)

## Разделение БД между 3 дисками

Определим производительность одного виртуального диска
```
fio --randrepeat=1 --ioengine=libaio --direct=1 --gtod_reduce=1 --name=fiotest --filename=testfio --bs=4k --iodepth=64 --size=2G --readwrite=randrw --rwmixread=75

  read: IOPS=78.0k, BW=305MiB/s (320MB/s)(6141MiB/20152msec)
  write: IOPS=26.1k, BW=102MiB/s (107MB/s)(2051MiB/20152msec); 0 zone resets
```

Установим ограничение IO в гипервизоре на чтение 30 MB/s и на запись 10 MB/s для теста с тремя дисками
```
fio --randrepeat=1 --ioengine=libaio --direct=1 --gtod_reduce=1 --name=fiotest --filename=testfio --bs=4k --iodepth=64 --size=2G --readwrite=randrw --rwmixread=75

  read: IOPS=7746, BW=30.3MiB/s (31.7MB/s)(1534MiB/50698msec)
  write: IOPS=2595, BW=10.1MiB/s (10.6MB/s)(514MiB/50698msec); 0 zone resets
```

Добавим 2 диска с ограничением IO, создадим разделы fdisk и ФС mkfs.ext4

Установим 12 ядер ЦПУ, чтобы не упираться в процессор, изменим настройки pg

### Всё на одном диске sda

```
random_page_cost = 1
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_maintenance_workers = 2
max_parallel_workers = 4
```
```
$ pgbench -c 300 -j 2 -T 30 -f ~/workload_read.sql -U postgres thai -n | grep -E 'transaction|tps'
transaction type: /var/lib/postgresql/workload_read.sql
number of transactions actually processed: 2406150
number of failed transactions: 0 (0.000%)
tps = 80683.060581 (without initial connection time)

CPU | sys     284%  | user    911%  |               | irq       0%  |               | idle      2%  | wait      0% |               |  steal     4% |  guest     0% |

$ pgbench -c 300 -j 2 -T 30 -f ~/workload_write.sql -U postgres thai 2>/dev/null | grep -E 'transaction|tps'
transaction type: /var/lib/postgresql/workload_write.sql
number of transactions actually processed: 580203
number of failed transactions: 0 (0.000%)
tps = 19528.547883 (without initial connection time)

DSK |          sda  | busy     91%  | read       0  | write   2182  | discrd     0  | KiB/r      0  | KiB/w     47 |  KiB/d      0 |  MBr/s    0.0 |  MBw/s   10.0 |
```

### Перенос wal файлов на sdb

- Остановить pg
- Перенести /var/lib/postgresql/15/main/pg_wal на новый диск
- Перемонтировать новый диск в /var/lib/postgresql/15/main/pg_wal
- Проверить владельца каталога и файлов - postgres postgres
- Запустить pg

```
$ pgbench -c 300 -j 2 -T 30 -f ~/workload_read.sql -U postgres thai -n | grep -E 'transaction|tps'
transaction type: /var/lib/postgresql/workload_read.sql
number of transactions actually processed: 2387157
number of failed transactions: 0 (0.000%)
tps = 80128.230303 (without initial connection time)

CPU | sys     294%  | user    845%  |               | irq       0%  |               | idle     58%  | wait      0% |               |  steal     4% |  guest     0% |

$ pgbench -c 300 -j 2 -T 30 -f ~/workload_write.sql -U postgres thai 2>/dev/null | grep -E 'transaction|tps'
transaction type: /var/lib/postgresql/workload_write.sql
number of transactions actually processed: 695923
number of failed transactions: 0 (0.000%)
tps = 23425.543467 (without initial connection time)

DSK |          sdb  | busy     69%  | read       0  | write   6608  | discrd     0  | KiB/r      0  | KiB/w     16 |  KiB/d      0 |  MBr/s    0.0 |  MBw/s   10.3 |
DSK |          sda  | busy     18%  | read       0  | write     62  | discrd     0  | KiB/r      0  | KiB/w    651 |  KiB/d      0 |  MBr/s    0.0 |  MBw/s    3.9 |
```

### Перенос таблицы в тейблспейс на отдельный диск sdc

- Смонтировать новый диск
- Проверить владельца каталога и файлов - postgres postgres
- CREATE TABLESPACE tickets_tblspc LOCATION '/mnt/tbl';
- ALTER TABLE book.tickets SET TABLESPACE tickets_tblspc;
- ALTER INDEX tickets_pkey SET TABLESPACE tickets_tblspc;

```
\db+
                                    List of tablespaces
      Name      |  Owner   | Location | Access privileges | Options |  Size   | Description 
----------------+----------+----------+-------------------+---------+---------+-------------
 pg_default     | postgres |          |                   |         | 297 MB  | 
 pg_global      | postgres |          |                   |         | 531 kB  | 
 tickets_tblspc | postgres | /mnt/tbl |                   |         | 6881 MB | 

\d book.tickets
    Table "book.tickets"
Tablespace: "tickets_tblspc"

\d tickets_pkey
    Index "book.tickets_pkey"
Tablespace: "tickets_tblspc"
```
```
$ pgbench -c 300 -j 2 -T 30 -f ~/workload_read.sql -U postgres thai -n | grep -E 'transaction|tps'
transaction type: /var/lib/postgresql/workload_read.sql
number of transactions actually processed: 2369719
number of failed transactions: 0 (0.000%)
tps = 79708.536236 (without initial connection time)

$ pgbench -c 300 -j 2 -T 30 -f ~/workload_write.sql -U postgres thai 2>/dev/null | grep -E 'transaction|tps'
transaction type: /var/lib/postgresql/workload_write.sql
number of transactions actually processed: 691709
number of failed transactions: 0 (0.000%)
tps = 23279.919229 (without initial connection time)

DSK |          sdb  | busy     66%  | read       0  | write   6398  | discrd     0  | KiB/r      0  | KiB/w     16 |  KiB/d      0 |  MBr/s    0.0 |  MBw/s   10.3 |
DSK |          sdc  | busy     23%  | read       0  | write     52  | discrd     0  | KiB/r      0  | KiB/w    885 |  KiB/d      0 |  MBr/s    0.0 |  MBw/s    4.5 |
DSK |          sda  | busy      0%  | read       0  | write     21  | discrd     0  | KiB/r      0  | KiB/w    220 |  KiB/d      0 |  MBr/s    0.0 |  MBw/s    0.5 |
```

- ALTER INDEX tickets_pkey SET TABLESPACE pg_default;
```
$ pgbench -c 300 -j 2 -T 30 -f ~/workload_read.sql -U postgres thai -n | grep -E 'transaction|tps'
transaction type: /var/lib/postgresql/workload_read.sql
number of transactions actually processed: 2356285
number of failed transactions: 0 (0.000%)
tps = 79469.026292 (without initial connection time)

$ pgbench -c 300 -j 2 -T 30 -f ~/workload_write.sql -U postgres thai 2>/dev/null | grep -E 'transaction|tps'
transaction type: /var/lib/postgresql/workload_write.sql
number of transactions actually processed: 695412
number of failed transactions: 0 (0.000%)
tps = 23378.018285 (without initial connection time)
```

![](img/img2.png)

## Сравнение производительности COPY vs PGLOADER

Испольузется БД thai_small
```
apt install pgloader -y

psql -U postgres -d thai -c "COPY tickets TO STDOUT WITH CSV DELIMITER ',';" > /tmp/tickets.csv

thai=# COPY tickets TO '/tmp/tickets.csv' WITH (FORMAT CSV);
COPY 5185505
Time: 3541.762 ms (00:03.542)

thai=# TRUNCATE tickets;
thai=# COPY tickets FROM '/tmp/tickets.csv' WITH (FORMAT csv);
COPY 5185505
Time: 76629.993 ms (01:16.630)
```

PGLOADER без 'drop indexes'
```
pgloader --type csv                         \
         --with "fields terminated by ','"  \
         --with "truncate"                  \
         /tmp/tickets.csv                   \
         pgsql://postgres:123@localhost:5432/thai?tickets

2024-07-25T15:55:21.016000Z LOG pgloader version "3.6.7~devel"
2024-07-25T15:55:21.016000Z LOG Data errors in '/tmp/pgloader/'
2024-07-25T15:55:21.196001Z WARNING Target table "\"book\".\"tickets\"" has 1 indexes defined against it.
2024-07-25T15:55:21.196001Z WARNING That could impact loading performance badly.
2024-07-25T15:55:21.196001Z WARNING Consider the option 'drop indexes'.
2024-07-25T15:57:06.680624Z LOG report summary reset
             table name     errors       rows      bytes      total time
-----------------------  ---------  ---------  ---------  --------------
                  fetch          0          0                     0.004s
-----------------------  ---------  ---------  ---------  --------------
       "book"."tickets"          0    5185505   289.7 MB       1m45.377s
-----------------------  ---------  ---------  ---------  --------------
        Files Processed          0          1                     0.012s
COPY Threads Completion          0          2                  1m45.377s
-----------------------  ---------  ---------  ---------  --------------
      Total import time          ✓    5185505   289.7 MB       1m45.389s
```

PGLOADER с 'drop indexes'
```
pgloader --type csv                         \
         --with "fields terminated by ','"  \
         --with "truncate"                  \
         --with "drop indexes"              \
         /tmp/tickets.csv                   \
         pgsql://postgres:123@localhost:5432/thai?tickets

2024-07-25T15:58:28.012000Z LOG pgloader version "3.6.7~devel"
2024-07-25T15:58:28.016000Z LOG Data errors in '/tmp/pgloader/'
2024-07-25T16:00:13.580624Z LOG report summary reset
             table name     errors       rows      bytes      total time
-----------------------  ---------  ---------  ---------  --------------
                  fetch          0          0                     0.004s
-----------------------  ---------  ---------  ---------  --------------
       "book"."tickets"          0    5185505   289.7 MB       1m41.881s
-----------------------  ---------  ---------  ---------  --------------
        Files Processed          0          1                     0.012s
COPY Threads Completion          0          2                  1m41.881s
 Index Build Completion          0          0                     3.348s
         Create Indexes          0          1                     3.296s
            Constraints          0          1                     0.000s
-----------------------  ---------  ---------  ---------  --------------
      Total import time          ✓    5185505   289.7 MB       1m48.537s
```

Результат COPY - 01:16

Результат PGLOADER - 01:45 (+38%)

Если запускать PGLOADER без "truncate" и с "drop indexes", будет ошибка, нужно следить что параметры указаны правильно
```
QUERY: CREATE UNIQUE INDEX tickets_pkey ON book.tickets USING btree (id)
2024-07-25T15:44:25.436602Z ERROR PostgreSQL Database error 42704: index "tickets_pkey" does not exist
QUERY: ALTER TABLE "book"."tickets" ADD CONSTRAINT "tickets_pkey" PRIMARY KEY USING INDEX "tickets_pkey";

thai=# select * from tickets where id=20458;
  id   | fkride |      fio      |          contact          | fkseat 
-------+--------+---------------+---------------------------+--------
 20458 |  92000 | BELOVA EDUARD | {"phone": "+73238731834"} |      1
 20458 |  92000 | BELOVA EDUARD | {"phone": "+73238731834"} |      1
(2 rows)

grep -E ^20458, /tmp/tickets.csv 
20458,92000,BELOVA EDUARD,"{""phone"": ""+73238731834""}",1

ALTER TABLE ONLY book.tickets ADD CONSTRAINT tickets_pkey PRIMARY KEY (id);
ALTER TABLE "book"."tickets" ADD CONSTRAINT "tickets_pkey" PRIMARY KEY USING INDEX "tickets_pkey";
```

Выводы
- партиционирование может ускорить чтение данных
- перенос wal файлов на другой диск может ускорить запись
- перенос данных на другой диск теоретически также может ускорить чтение/запись в определенных случаях
- COPY быстрее чем PGLOADER

Ссылки:
- https://winitpro.ru/index.php/2020/04/30/test-iops-diskov-v-linux/
- https://www.postgresql.org/docs/current/progress-reporting.html
- https://pgloader.readthedocs.io/en/latest/index.html
