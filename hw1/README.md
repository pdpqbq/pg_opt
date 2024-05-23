# Первичная настройка ОС и PostgreSQL

## Конфигурация стенда

PostgreSQL запущен в докере на локальном хосте, 8 cpu, 32 ram, swap отключен
```
psql (16.2, server 16.3 (Debian 16.3-1.pgdg120+1))

pgbench (PostgreSQL) 16.2
```
Настройки по-умолчанию
```
max_worker_processes = 8
max_connections = 100
shared_buffers = 128MB
max_wal_size = 1GB
min_wal_size = 80MB
fsync = on
synchronous_commit = on
random_page_cost = 4
huge_pages = try
```
```
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
```
## Инициализация

```
pgbench -U pg -h localhost -d pg -i
```

## Тест 1 клиент 1 поток

```
pgbench -U pg -h localhost -d pg -T 15 -r

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 15 s
number of transactions actually processed: 4259
number of failed transactions: 0 (0.000%)
latency average = 3.520 ms
initial connection time = 8.113 ms
tps = 284.080411 (without initial connection time)
statement latencies in milliseconds and failures:
         0.009           0  \set aid random(1, 100000 * :scale)
         0.005           0  \set bid random(1, 1 * :scale)
         0.005           0  \set tid random(1, 10 * :scale)
         0.005           0  \set delta random(-5000, 5000)
         0.098           0  BEGIN;
         0.224           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         0.154           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         0.163           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         0.152           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.137           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         2.562           0  END;
```

## Тест 100 клиентов 8 потоков

```
pgbench -U pg -h localhost -d pg -T 15 -r -c 100 -j 8

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 8
maximum number of tries: 1
duration: 15 s
number of transactions actually processed: 4913
number of failed transactions: 0 (0.000%)
latency average = 300.921 ms
initial connection time = 514.856 ms
tps = 332.313531 (without initial connection time)
statement latencies in milliseconds and failures:
         0.016           0  \set aid random(1, 100000 * :scale)
         0.013           0  \set bid random(1, 1 * :scale)
         0.010           0  \set tid random(1, 10 * :scale)
         0.011           0  \set delta random(-5000, 5000)
         0.180           0  BEGIN;
         0.534           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         0.233           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
       268.447           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
        25.385           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.217           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         2.713           0  END;
```

## Тест 100 клиентов 8 потоков с опцией "establish new connection for each transaction"

```
pgbench -U pg -h localhost -d pg -T 15 -r -c 100 -j 8 -C

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 8
maximum number of tries: 1
duration: 15 s
number of transactions actually processed: 2289
number of failed transactions: 0 (0.000%)
latency average = 669.669 ms
average connection time = 15.407 ms
tps = 149.327469 (including reconnection times)
statement latencies in milliseconds and failures:
         0.033           0  \set aid random(1, 100000 * :scale)
         0.008           0  \set bid random(1, 1 * :scale)
         0.007           0  \set tid random(1, 10 * :scale)
         0.008           0  \set delta random(-5000, 5000)
        10.710           0  BEGIN;
         1.707           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         0.229           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
       567.166           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
        53.721           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.726           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         2.765           0  END;
```

## Новые настройки 

Используется конфигуратор https://pgconfigurator.cybertec.at

```
shared_buffers = '8192 MB'
work_mem = '32 MB'
maintenance_work_mem = '420 MB'
effective_cache_size = '22 GB'
effective_io_concurrency = 100
random_page_cost = 1.25
```

## Тест 100 клиентов 8 потоков

```
pgbench -U pg -h localhost -d pg -T 15 -r -c 100 -j 8

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 8
maximum number of tries: 1
duration: 15 s
number of transactions actually processed: 4868
number of failed transactions: 0 (0.000%)
latency average = 303.946 ms
initial connection time = 536.971 ms
tps = 329.005550 (without initial connection time)
statement latencies in milliseconds and failures:
         0.017           0  \set aid random(1, 100000 * :scale)
         0.012           0  \set bid random(1, 1 * :scale)
         0.011           0  \set tid random(1, 10 * :scale)
         0.011           0  \set delta random(-5000, 5000)
         0.216           0  BEGIN;
         0.691           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         0.213           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
       270.953           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
        25.729           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.218           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         2.720           0  END;
```

Видим, что изменение параметров памяти не повлияло на TPS. По данным утилиты atop, диск загружен на 101%

```
DSK |      nvme0n1  | busy    101%  | read       0  | write   7238
```

## Изменим настройки на максимальную производительность работы с диском без учета D в ACID

```
synchronous_commit = off
```
```
DSK |      nvme0n1  | busy      7%  | read       0  | write    325
```
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 8
maximum number of tries: 1
duration: 15 s
number of transactions actually processed: 50035
number of failed transactions: 0 (0.000%)
latency average = 29.093 ms
initial connection time = 507.600 ms
tps = 3437.217658 (without initial connection time)
statement latencies in milliseconds and failures:
         0.008           0  \set aid random(1, 100000 * :scale)
         0.006           0  \set bid random(1, 1 * :scale)
         0.005           0  \set tid random(1, 10 * :scale)
         0.005           0  \set delta random(-5000, 5000)
         0.065           0  BEGIN;
         0.213           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         0.109           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
        26.045           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         2.330           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.105           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         0.125           0  END;
```
```
fsync = off
full_page_writes = off
```
```
DSK |      nvme0n1  | busy      1%  | read       0  | write    193  |
```
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 8
maximum number of tries: 1
duration: 15 s
number of transactions actually processed: 47568
number of failed transactions: 0 (0.000%)
latency average = 30.640 ms
initial connection time = 508.102 ms
tps = 3263.727994 (without initial connection time)
statement latencies in milliseconds and failures:
         0.009           0  \set aid random(1, 100000 * :scale)
         0.006           0  \set bid random(1, 1 * :scale)
         0.006           0  \set tid random(1, 10 * :scale)
         0.006           0  \set delta random(-5000, 5000)
         0.070           0  BEGIN;
         0.218           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         0.120           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
        27.386           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         2.467           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.112           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         0.136           0  END;
```

## Тест 20 клиентов 8 потоков

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 8
maximum number of tries: 1
duration: 15 s
number of transactions actually processed: 69931
number of failed transactions: 0 (0.000%)
latency average = 4.264 ms
initial connection time = 105.940 ms
tps = 4690.736448 (without initial connection time)
statement latencies in milliseconds and failures:
         0.008           0  \set aid random(1, 100000 * :scale)
         0.007           0  \set bid random(1, 1 * :scale)
         0.006           0  \set tid random(1, 10 * :scale)
         0.007           0  \set delta random(-5000, 5000)
         0.046           0  BEGIN;
         0.127           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         0.095           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         2.494           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         1.288           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.090           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         0.082           0  END;
```

## Тест 10 клиентов 4 потока
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 4
maximum number of tries: 1
duration: 15 s
number of transactions actually processed: 73112
number of failed transactions: 0 (0.000%)
latency average = 2.044 ms
initial connection time = 63.135 ms
tps = 4892.217579 (without initial connection time)
statement latencies in milliseconds and failures:
         0.007           0  \set aid random(1, 100000 * :scale)
         0.006           0  \set bid random(1, 1 * :scale)
         0.006           0  \set tid random(1, 10 * :scale)
         0.006           0  \set delta random(-5000, 5000)
         0.047           0  BEGIN;
         0.122           0  UPDATE pgbench_accounts SET abalance = abalance + :delta WHERE aid = :aid;
         0.096           0  SELECT abalance FROM pgbench_accounts WHERE aid = :aid;
         0.711           0  UPDATE pgbench_tellers SET tbalance = tbalance + :delta WHERE tid = :tid;
         0.871           0  UPDATE pgbench_branches SET bbalance = bbalance + :delta WHERE bid = :bid;
         0.088           0  INSERT INTO pgbench_history (tid, bid, aid, delta, mtime) VALUES (:tid, :bid, :aid, :delta, CURRENT_TIMESTAMP);
         0.073           0  END;
```

Уменьшение числа клиентов снижает задержки на операции UPDATE и даёт больше TPS

## Выводы

 - открытие нового подключения - дорогая операция
 - базу данных и нагрузочный тест нужно запускать с разных машин
 - уменьшение надежности системы хранения приводит к значительному увеличению производительности
 - данный тест при таком объеме данных и конфигурации стенда не является показательным для оценки влияния параметров настройки памяти на производительность