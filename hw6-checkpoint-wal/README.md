# Тюнинг checkpoint и WAL

Создать кластер ПГ и провести исследование с разными параметрами WAL + checkpoint в вариантах производительность VS надежность

## Стенд

VM Debian 12

### Установка PostgreSQL 16
```
apt -y install curl
curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list
apt update
apt -y install postgresql-16
```

### Вывод информации о кластере
```
/usr/lib/postgresql/16/bin/pg_controldata /var/lib/postgresql/16/main/
...
Latest checkpoint location:           0/152AB28
Latest checkpoint's REDO location:    0/152AAF0
Latest checkpoint's REDO WAL file:    000000010000000000000001
Latest checkpoint's TimeLineID:       1
Latest checkpoint's PrevTimeLineID:   1
Latest checkpoint's full_page_writes: on
Latest checkpoint's NextXID:          0:741
Latest checkpoint's NextOID:          24576
Latest checkpoint's NextMultiXactId:  1
Latest checkpoint's NextMultiOffset:  0
Latest checkpoint's oldestXID:        722
Latest checkpoint's oldestXID's DB:   1
Latest checkpoint's oldestActiveXID:  741
Latest checkpoint's oldestMultiXid:   1
Latest checkpoint's oldestMulti's DB: 1
Latest checkpoint's oldestCommitTsXid:0
Latest checkpoint's newestCommitTsXid:0
Time of latest checkpoint:            Wed 07 Aug 2024 08:50:42 AM CDT
wal_level setting:                    replica
wal_log_hints setting:                off
max_connections setting:              100
max_worker_processes setting:         8
max_wal_senders setting:              10
max_prepared_xacts setting:           0
max_locks_per_xact setting:           64
track_commit_timestamp setting:       off
Maximum data alignment:               8
Database block size:                  8192
Blocks per segment of large relation: 131072
WAL block size:                       8192
Bytes per WAL segment:                16777216
Maximum length of identifiers:        64
Maximum columns in an index:          32
Maximum size of a TOAST chunk:        1996
Size of a large-object chunk:         2048
Data page checksum version:           0
```

## Влияние wal_level на размер информации в wal файлах

Методика тестирования:
- получаем текущий lsn
- выполняем различные операции
- получаем текущий lsn
- проверяем через pg_wal_lsn_diff
- проверяем через pg_stat_statements
- проверяем через pg_waldump
```
TRUNCATE TABLE t;
VACUUM FULL;
SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());
SELECT pg_stat_statements_reset();
INSERT INTO t SELECT gen_random_uuid() FROM generate_series(1,1000);
UPDATE t SET u = gen_random_uuid();
CREATE TABLE q (t text);
ALTER TABLE t ADD COLUMN t text;
ALTER TABLE t DROP COLUMN t;
DROP TABLE q;
SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());

SELECT pg_wal_lsn_diff('0/3E157C88', '0/3E129518');
SELECT query, calls, rows, wal_records, wal_bytes FROM pg_stat_statements WHERE query NOT LIKE 'SELECT%';
SELECT sum(wal_bytes) FROM pg_stat_statements;
\! /usr/lib/postgresql/16/bin/pg_waldump -p /var/lib/postgresql/16/main/pg_wal -s 0/3E129518 -e 0/3E157C88 -z
```

Подготовка - включение pg_stat_statements и создание таблицы
```
postgres=# SELECT * FROM pg_available_extensions WHERE name = 'pg_stat_statements';
        name        | default_version | installed_version |                                comment                                 
--------------------+-----------------+-------------------+------------------------------------------------------------------------
 pg_stat_statements | 1.10            |                   | track planning and execution statistics of all SQL statements executed
(1 row)

postgres=# CREATE EXTENSION pg_stat_statements;
CREATE EXTENSION

postgres=# SELECT * FROM pg_available_extensions WHERE name = 'pg_stat_statements';
        name        | default_version | installed_version |                                comment                                 
--------------------+-----------------+-------------------+------------------------------------------------------------------------
 pg_stat_statements | 1.10            | 1.10              | track planning and execution statistics of all SQL statements executed
(1 row)

alter system set shared_preload_libraries = 'pg_stat_statements';

CREATE TABLE t (u uuid);
```

### wal_level = 'minimal'

Здесь также нужно установить max_wal_senders = 0
```
psql -c "alter system set wal_level = 'minimal'" && \
pg_ctlcluster 16 main stop && pg_ctlcluster 16 main start && \
/usr/lib/postgresql/16/bin/pg_controldata /var/lib/postgresql/16/main | grep wal_level

FATAL:  WAL streaming (max_wal_senders > 0) requires wal_level "replica" or "logical"
pg_ctl: could not start server
Examine the log output.

echo "max_wal_senders = 0" >> /etc/postgresql/16/main/postgresql.conf
pg_ctlcluster 16 main start
/usr/lib/postgresql/16/bin/pg_controldata /var/lib/postgresql/16/main | grep wal_level

wal_level setting:                    minimal
```
```
 pg_wal_lsn_diff 
-----------------
          259784
(1 row)

                               query                                | calls | rows | wal_records | wal_bytes 
--------------------------------------------------------------------+-------+------+-------------+-----------
 ALTER TABLE t ADD COLUMN t text                                    |     1 |    0 |          53 |      5758
 ALTER TABLE t DROP COLUMN t                                        |     1 |    0 |           4 |       397
 CREATE TABLE q (t text)                                            |     1 |    0 |          87 |      9341
 DROP TABLE q                                                       |     1 |    0 |          31 |      1695
 INSERT INTO t SELECT gen_random_uuid() FROM generate_series($1,$2) |     1 | 1000 |        1000 |     71000
 UPDATE t SET u = gen_random_uuid()                                 |     1 | 1000 |        2000 |    144000
(6 rows)

  sum   
--------
 232191
(1 row)

WAL statistics between 0/1CDF6F0 and 0/1D1EDB8:
Type                                           N      (%)          Record size      (%)             FPI size      (%)        Combined size      (%)
----                                           -      ---          -----------      ---             --------      ---        -------------      ---
XLOG                                           2 (  0.06)                   98 (  0.04)                16384 (100.00)                16482 (  6.60)
Transaction                                    6 (  0.19)                  288 (  0.12)                    0 (  0.00)                  288 (  0.12)
Storage                                        5 (  0.16)                  210 (  0.09)                    0 (  0.00)                  210 (  0.08)
CLOG                                           0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Database                                       0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Tablespace                                     0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
MultiXact                                      0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
RelMap                                         0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Standby                                        0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Heap2                                         29 (  0.91)                 5854 (  2.51)                    0 (  0.00)                 5854 (  2.34)
Heap                                        3048 ( 95.46)               219811 ( 94.20)                    0 (  0.00)               219811 ( 88.02)
Btree                                        103 (  3.23)                 7072 (  3.03)                    0 (  0.00)                 7072 (  2.83)
Hash                                           0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Gin                                            0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Gist                                           0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Sequence                                       0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
SPGist                                         0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
BRIN                                           0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
CommitTs                                       0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
ReplicationOrigin                              0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Generic                                        0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
LogicalMessage                                 0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
                                        --------                      --------                      --------                      --------
Total                                       3193                        233333 [93.44%]                16384 [6.56%]                249717 [100%]
```

### wal_level = 'replica'

```
psql -c "alter system set wal_level = 'replica'" && \
pg_ctlcluster 16 main stop && pg_ctlcluster 16 main start && \
/usr/lib/postgresql/16/bin/pg_controldata /var/lib/postgresql/16/main | grep wal_level

wal_level setting:                    replica
```
```
 pg_wal_lsn_diff 
-----------------
          250584
(1 row)

                               query                                | calls | rows | wal_records | wal_bytes 
--------------------------------------------------------------------+-------+------+-------------+-----------
 ALTER TABLE t ADD COLUMN t text                                    |     1 |    0 |          60 |      6368
 ALTER TABLE t DROP COLUMN t                                        |     1 |    0 |           5 |       435
 CREATE TABLE q (t text)                                            |     1 |    0 |          90 |     13237
 DROP TABLE q                                                       |     1 |    0 |          34 |      1831
 INSERT INTO t SELECT gen_random_uuid() FROM generate_series($1,$2) |     1 | 1000 |        1000 |     71000
 UPDATE t SET u = gen_random_uuid()                                 |     1 | 1000 |        2000 |    144000
(6 rows)

  sum   
--------
 236871
(1 row)

WAL statistics between 0/22B1C40 and 0/22EEF18:
Type                                           N      (%)          Record size      (%)             FPI size      (%)        Combined size      (%)
----                                           -      ---          -----------      ---             --------      ---        -------------      ---
XLOG                                           2 (  0.06)                   98 (  0.04)                  176 (100.00)                  274 (  0.11)
Transaction                                    6 (  0.19)                 3061 (  1.27)                    0 (  0.00)                 3061 (  1.27)
Storage                                        5 (  0.16)                  210 (  0.09)                    0 (  0.00)                  210 (  0.09)
CLOG                                           0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Database                                       0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Tablespace                                     0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
MultiXact                                      0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
RelMap                                         0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Standby                                        9 (  0.28)                  394 (  0.16)                    0 (  0.00)                  394 (  0.16)
Heap2                                         24 (  0.75)                 5497 (  2.29)                    0 (  0.00)                 5497 (  2.29)
Heap                                        3049 ( 95.22)               220008 ( 91.55)                    0 (  0.00)               220008 ( 91.48)
Btree                                        107 (  3.34)                11056 (  4.60)                    0 (  0.00)                11056 (  4.60)
Hash                                           0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Gin                                            0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Gist                                           0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Sequence                                       0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
SPGist                                         0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
BRIN                                           0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
CommitTs                                       0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
ReplicationOrigin                              0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Generic                                        0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
LogicalMessage                                 0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
                                        --------                      --------                      --------                      --------
Total                                       3202                        240324 [99.93%]                  176 [0.07%]                240500 [100%]
```

### wal_level = 'logical'

```
psql -c "alter system set wal_level = 'logical'" && \
pg_ctlcluster 16 main stop && pg_ctlcluster 16 main start && \
/usr/lib/postgresql/16/bin/pg_controldata /var/lib/postgresql/16/main | grep wal_level

wal_level setting:                    logical
```
```
 pg_wal_lsn_diff 
-----------------
          255736
(1 row)

                               query                                | calls | rows | wal_records | wal_bytes 
--------------------------------------------------------------------+-------+------+-------------+-----------
 ALTER TABLE t ADD COLUMN t text                                    |     1 |    0 |          84 |      8996
 ALTER TABLE t DROP COLUMN t                                        |     1 |    0 |           7 |       592
 CREATE TABLE q (t text)                                            |     1 |    0 |         126 |     12529
 DROP TABLE q                                                       |     1 |    0 |          69 |      4689
 INSERT INTO t SELECT gen_random_uuid() FROM generate_series($1,$2) |     1 | 1000 |        1000 |     71000
 UPDATE t SET u = gen_random_uuid()                                 |     1 | 1000 |        2000 |    144000
(6 rows)

  sum   
--------
 241806
(1 row)

WAL statistics between 0/28FE5E0 and 0/293CCD8:
Type                                           N      (%)          Record size      (%)             FPI size      (%)        Combined size      (%)
----                                           -      ---          -----------      ---             --------      ---        -------------      ---
XLOG                                           2 (  0.06)                   98 (  0.04)                  176 (100.00)                  274 (  0.11)
Transaction                                   23 (  0.70)                 6330 (  2.58)                    0 (  0.00)                 6330 (  2.58)
Storage                                        5 (  0.15)                  210 (  0.09)                    0 (  0.00)                  210 (  0.09)
CLOG                                           0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Database                                       0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Tablespace                                     0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
MultiXact                                      0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
RelMap                                         0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Standby                                        9 (  0.27)                  394 (  0.16)                    0 (  0.00)                  394 (  0.16)
Heap2                                        106 (  3.22)                10389 (  4.24)                    0 (  0.00)                10389 (  4.24)
Heap                                        3048 ( 92.45)               219811 ( 89.68)                    0 (  0.00)               219811 ( 89.61)
Btree                                        104 (  3.15)                 7880 (  3.21)                    0 (  0.00)                 7880 (  3.21)
Hash                                           0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Gin                                            0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Gist                                           0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Sequence                                       0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
SPGist                                         0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
BRIN                                           0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
CommitTs                                       0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
ReplicationOrigin                              0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
Generic                                        0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
LogicalMessage                                 0 (  0.00)                    0 (  0.00)                    0 (  0.00)                    0 (  0.00)
                                        --------                      --------                      --------                      --------
Total                                       3297                        245112 [99.93%]                  176 [0.07%]                245288 [100%]
```

### Размер wal при разном уровне wal_level и разных методах подсчета

| | lsn_diff | pg_stat | waldump |
| ------- | ------- | ------- | ------- |
| minimal | 259 784 | 232 191 | 249 717 |
| replica | 250 584 | 236 871 | 240 500 |
| logical | 255 736 | 241 806 | 245 288 |

## Восстановление кластера при разных значениях параметров

Методика:
- очищаем wal
- делаем update в течение 4 минут
- делаем kill главного процесса
- запускаем кластер и засекаем время запуска
- параллельно смотрим объем wal и лог postgresql
```
watch du -sh /var/lib/postgresql/16/main/pg_wal/
tail -f /var/log/postgresql/postgresql-16-main.log

cat > write.sql <<EOF
update t set u = gen_random_uuid();
EOF

pg_ctlcluster 16 main stop
/usr/lib/postgresql/16/bin/pg_resetwal -f -D /var/lib/postgresql/16/main
pg_ctlcluster 16 main start

psql -c "vacuum full"
psql -c "checkpoint"
pgbench -c 1 -j 4 -T 240 -f ~/write.sql -U postgres -n
kill -9 `cat /run/postgresql/16-main.pid`

pg_ctlcluster 16 main start
```

### Тест 1 - больше checkpoint_timeout и max_wal_size, full_page_writes on
```
checkpoint_timeout   5min
max_wal_size         10GB
full_page_writes     on
wal_level            logical
```

Накопилось 7.1G wal, чекпойнта не было, запуск 49 сек
```
LOG:  database system was not properly shut down; automatic recovery in progress
...
LOG:  redo done at 3/41E738F0 system usage: CPU: user: 44.19 s, system: 5.16 s, elapsed: 49.73 s
```

### Тест 2 - меньше checkpoint_timeout и max_wal_size, full_page_writes off
```
checkpoint_timeout   1min
max_wal_size         1GB
full_page_writes     off
wal_level            logical
```
При достижении размера wal 1G происходит регулярный чекпойнт, запуск 5 сек
```
LOG:  database system was not properly shut down; automatic recovery in progress
...
LOG:  redo done at 6/CCEF00B0 system usage: CPU: user: 4.79 s, system: 0.34 s, elapsed: 5.13 s
```

### Тест 3 - меньше checkpoint_timeout и max_wal_size, full_page_writes on
```
checkpoint_timeout   1min
max_wal_size         1GB
full_page_writes     on
wal_level            logical
```
При достижении размера wal 1G происходит регулярный чекпойнт, запуск <5 сек
```
LOG:  database system was not properly shut down; automatic recovery in progress
...
LOG:  redo done at 8/CF77C798 system usage: CPU: user: 4.16 s, system: 0.31 s, elapsed: 4.48 s
```

### Описание параметров

- wal_buffers

Количество памяти используемое в SHARED MEMORY для ведения транзакционных логов. При доступной памяти 1-4GB рекомендуется устанавливать 256-1024kb. Этот параметр стоит увеличивать в системах с большим количеством модификаций таблиц базы данных.

- checkpoint_segments

Oпределяет количество сегментов (каждый по 16 МБ) лога транзакций между контрольными точками.  Для баз данных со множеством модифицирующих данные транзакций рекомендуется увеличение этого параметра. Критерием достаточности количества сегментов является отсутствие в логе предупреждений (warning) о том, что контрольные точки происходят слишком часто.

- full_page_writes

Включение этого параметра гарантирует корректное восстановление, ценой увеличения записываемых данных в журнал транзакций. Отключение этого параметра ускоряет работу, но может привести к повреждению базы данных в случае системного сбоя или отключения питания. 

Reducing checkpoint_timeout and/or max_wal_size causes checkpoints to occur more often. This allows faster after-crash recovery, since less work will need to be redone.However, one must balance this against the increased cost of flushing dirty data pages more often. If full_page_writes is set (as is the default), there is another factor to consider. To ensure data page consistency, the first modification of a data page after each checkpoint results in logging the entire page content. In that case, a smaller checkpoint interval increases the volume of output to the WAL, partially negating the goal of using a smaller interval, and in any case causing more disk I/O.

## Параметр wal_sync_method
 
Значение по умолчанию wal_sync_method = fdatasync даёт лучшую производительность; все значения, кроме fsync_writethrough, одинаковы по надежности (по документации)

All the options should be the same in terms of reliability, with the exception of fsync_writethrough
```
/usr/lib/postgresql/16/bin/pg_test_fsync
...
Compare file sync methods using two 8kB writes:
(in wal_sync_method preference order, except fdatasync is Linux's default)
        open_datasync                       621.502 ops/sec    1609 usecs/op
        fdatasync                           846.569 ops/sec    1181 usecs/op
        fsync                               779.939 ops/sec    1282 usecs/op
        fsync_writethrough                              n/a
        open_sync                           468.570 ops/sec    2134 usecs/op
...
```

## Сжатие wal файлов

Очищаем таблицу
```
TRUNCATE TABLE t;
VACUUM FULL;
```

Во втором окне запускаем iotop (-a show accumulated I/O instead of bandwidth)
```
iotop -u postgres -a -k
```

Делаем вставку
```
INSERT INTO t SELECT gen_random_uuid() FROM generate_series(1,1000000);
CHECKPOINT;
```

Сжатие выключено, wal_compression = off, wal_level = logical
```
    TID  PRIO  USER     DISK READ DISK WRITE>    COMMAND                                                                                                      
  13962 be/4 postgres      0.00 K  96008.00 K postgres: 16/main: postgres postgres [local] idle
  13931 be/4 postgres      0.00 K  33352.00 K postgres: 16/main: walwriter
  13928 be/4 postgres      0.00 K    888.00 K postgres: 16/main: checkpointer
```

Сжатие включено, wal_compression = pglz, wal_level = logical
```
    TID  PRIO  USER     DISK READ DISK WRITE>    COMMAND                                                                                                      
  13996 be/4 postgres      0.00 K 119032.00 K postgres: 16/main: postgres postgres [local] idle
  13983 be/4 postgres      0.00 K  10336.00 K postgres: 16/main: walwriter
  13980 be/4 postgres      0.00 K    888.00 K postgres: 16/main: checkpointer
```

Видим, что при включении сжатия уменьшился объем данных, записанных процессом walwriter

Выводы:
- при повышении уровня логирования wal_level увеличивается количество информации в части DDL
- при увеличении периода чекпойнтов восстановление кластера из журналов идёт дольше
- сжатие wal позволяет сэкономить место на диске за счет использования процессора

Ссылки:
- https://www.postgresql.org/docs/current/wal-configuration.html
- https://www.postgresql.org/docs/current/pgtestfsync.html
- https://www.postgresql.org/docs/current/wal-reliability.html
- https://postgrespro.ru/docs/postgrespro/9.5/pgtestfsync
- https://postgrespro.com/blog/pgsql/5967974
- https://pgpedia.info/p/pg_switch_wal.html
