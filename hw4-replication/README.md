# Репликация

### Конфигурация стенда

- Debian 12

- PostgreSQL 15

- Мастер и две реплики на разных ВМ (LXC) по 4 Гб памяти

- Начальные настройки одинаковые:
```
/etc/postgresql/15/main/postgresql.conf

listen_addresses = '*'
shared_buffers=1GB
work_mem=16MB

/etc/postgresql/15/main/pg_hba.conf
host	replication	repuser		192.168.1.0/24		scram-sha-256
```

### Мастер

```
create user repuser with replication login password 'password';
```

### Реп1

```
systemctl stop postgresql

(postgres) rm -r /var/lib/postgresql/15/main/
(postgres) pg_basebackup -h 192.168.1.51 -U repuser -X stream -C -S repslot1 -v -R -W -D /var/lib/postgresql/15/main/

systemctl start postgresql
```

### Мастер

Проверка
```
select slot_name, slot_type from pg_replication_slots;
 slot_name | slot_type 
-----------+-----------
 repslot1  | physical

create table t(i int);
insert into t values (1);
```

### Реп1

Репликация работает
```
select * from t;
 i 
---
 1
(1 row)
```

### Реп2

Аналогично для второй реплики, создается свой слот т.к. первый использовать нельзя
```
systemctl stop postgresql

(postgres) rm -r /var/lib/postgresql/15/main/
(postgres) pg_basebackup -h 192.168.1.51 -U repuser -X stream -C -S repslot2 -v -R -W -D /var/lib/postgresql/15/main/

systemctl start postgresql

select * from t;
 i 
---
 1
(1 row)
```

# Мастер и разные режимы репликации

По умолчанию асинхронная репликация, т.к. не указаны имена в synchronous_standby_names

Имена реплик задаются в параметре cluster_name
```
select slot_name, slot_type from pg_replication_slots;
 slot_name | slot_type 
-----------+-----------
 repslot1  | physical
 repslot2  | physical

\dconfig synchron*
List of configuration parameters
         Parameter         | Value 
---------------------------+-------
 synchronize_seqscans      | on
 synchronous_commit        | on
 synchronous_standby_names | 

select application_name, client_addr, sync_state from pg_stat_replication;
 application_name |  client_addr | sync_state 
------------------+---------------+------------
 15/main          | 192.168.1.52 | async
 15/main          | 192.168.1.53 | async
```

| synchronous_commit setting | local durable commit | standby durable commit after PG crash | standby durable commit after OS crash | standby query consistency |
|--------------|---|---|---|---|
| remote_apply | • | • | • | • |
| on           | • | • | • |   |
| remote_write | • | • |   |   |
| local        | • |   |   |   |
| off          |   |   |   |   |

```
set synchronous_standby_names='rep1'; select pg_reload_conf();
```
Инсерт зависает, т.к. не задан cluster_name на реплике

### Реп1

Установим cluster_name
```
\dconfig cluster_name 
List of configuration parameters
  Parameter   |  Value  
--------------+---------
 cluster_name | 15/main
(1 row)

\dconfig app*
List of configuration parameters
    Parameter     | Value 
------------------+-------
 application_name | psql
(1 row)

alter system set cluster_name ='rep1';
restart 
```

### Мастер

Инсерт работает, появилась синхронная реплика
```
select application_name, client_addr, sync_state from pg_stat_replication;
 application_name | client_addr  | sync_state 
------------------+--------------+------------
 rep1             | 192.168.1.52 | sync
 15/main          | 192.168.1.53 | async
```

### Реп2

Установим cluster_name
```
alter system set cluster_name='rep2';
restart
```

### Мастер

Возможны разные варианты
```
alter system set synchronous_standby_names ='FIRST 1 (*)';
select pg_reload_conf();

select application_name, client_addr, sync_state from pg_stat_replication;
 application_name |  client_addr  | sync_state 
------------------+---------------+------------
 rep2             | 192.168.1.53 | potential
 rep1             | 192.168.1.52 | sync

alter system set synchronous_standby_names='FIRST 2 (rep1,rep2)';
select pg_reload_conf();

select application_name, client_addr, sync_state from pg_stat_replication;
 application_name | client_addr  | sync_state 
------------------+--------------+------------
 rep1             | 192.168.1.52 | sync
 rep2             | 192.168.1.53 | sync

alter system set synchronous_standby_names ='ANY 1 (rep1,rep2)';
select pg_reload_conf();

select application_name, client_addr, sync_state from pg_stat_replication;
 application_name |  client_addr  | sync_state 
------------------+---------------+------------
 rep2             | 192.168.1.53 | quorum
 rep1             | 192.168.1.52 | quorum

alter system set synchronous_standby_names ='ANY 2 (*)';
select pg_reload_conf();

select application_name, client_addr, sync_state from pg_stat_replication;
 application_name |  client_addr  | sync_state 
------------------+---------------+------------
 rep2             | 192.168.1.53 | quorum
 rep1             | 192.168.1.52 | quorum
```

# Каскадная реплика

### Реп2

Будет каскадной репликой
```
stop
rm -r /var/lib/postgresql/15/main
pg_basebackup -h 192.168.1.52 -U repuser -X stream -C -S repslot3 -v -R -W -D /var/lib/postgresql/15/main/
start
alter system set cluster_name='rep2';
restart
```

### Мастер

Есть синхронная реплика
```
select application_name, client_addr, sync_state from pg_stat_replication;
 application_name | client_addr  | sync_state 
------------------+--------------+------------
 rep1             | 192.168.1.52 | sync
```

### Реп1

Каскадная реплика может быть только асинхронной
```
select application_name, client_addr, sync_state from pg_stat_replication;
 application_name | client_addr  | sync_state 
------------------+--------------+------------
 rep2             | 192.168.1.53 | async
```

# Тестирование скорости различных конфигураций

```
create user bench with password 'password';
grant all on schema public to bench;

pg_hba.conf
host    all             bench           192.168.1.0/24          trust

pgbench -i -U bench -h 192.168.1.51 -d postgres
pgbench -P 1 -c 10 -T 10 -j 4 -U bench -h 192.168.1.51 -p 5432 -d postgres -r

alter system set synchronous_commit='off'; select pg_reload_conf();
alter system set synchronous_commit='local'; select pg_reload_conf();
alter system set synchronous_commit='remote_write'; select pg_reload_conf();
alter system set synchronous_commit='on'; select pg_reload_conf();
alter system set synchronous_commit='remote_apply'; select pg_reload_conf();
```

2A - две асинхронных реплики, synchronous_standby_names=''

1S+1A - синхронная и асинхронная, synchronous_standby_names='rep1'

2S ALL - две синхронные, synchronous_standby_names='FIRST 2 (\*)'

2S ANY - две синхронные, кворум, synchronous_standby_names='ANY 1 (\*)'

A+C - асинхронная и каскадная

S+C - синхронная и каскадная

Результаты примерно одинаковы для двух групп комбинаций - с синхронной репликой и без

Здесь pgbench запускается как удаленный клиент БД по сети

|              | 2A  | 1S+1A | 2S ALL | 2S ANY | A+C | S+C |
|---           |---  |---    |---     |---     |---  |---  |
| remote_apply | 190 | 150   | 145    | 150    | 195 | 150 |
| on           | 190 | 145   | 145    | 150    | 200 | 150 |
| remote_write | 190 | 180   | 175    | 190    | 200 | 190 |
| local        | 205 | 190   | 190    | 190    | 195 | 210 |
| off          | 210 | 215   | 220    | 220    | 210 | 220 |

Здесь pgbench запускается на мастере, влияние synchronous_commit более заметно

|              | 2A   | 1S+1A | 2S ALL | 2S ANY | A+C | S+C |
|---           |---   |---    |---     |---     |---  |---  |
| remote_apply | 720  | 360   | 360    | ---    | --- | --- |
| on           | 670  | 360   | 320    | ---    | --- | --- |
| remote_write | 720  | 570   | 600    | ---    | --- | --- |
| local        | 720  | 790   | 720    | ---    | --- | --- |
| off          | 1890 | 1860  | 1880   | ---    | --- | --- |

# Исследование влияния параметра hot_standby_feedback

Создаем таблицы, объем подбирается по месту, чтобы аналитический запрос работал около минуты
```
drop table test1;
drop table test2;
create table test1 (u1 uuid, u2 uuid, u3 uuid, u4 uuid, u5 uuid);
create table test2 (u1 uuid, u2 uuid, u3 uuid, u4 uuid, u5 uuid);

CREATE OR REPLACE PROCEDURE trtest()
LANGUAGE plpgsql
AS $$
DECLARE
r bigint;
BEGIN
for r in 1..300000 loop
insert into test1 (u1,u2,u3,u4,u5) values (
gen_random_uuid(),
gen_random_uuid(),
gen_random_uuid(),
gen_random_uuid(),
gen_random_uuid()
);
insert into test2 (u1,u2,u3,u4,u5) values (
gen_random_uuid(),
gen_random_uuid(),
gen_random_uuid(),
gen_random_uuid(),
gen_random_uuid()
);
END LOOP;
COMMIT;
END;
$$;

call trtest();
```
Запускаем на реплике аналитический запрос
```
select count(*) from test1 t1 join test2 t2 on t1.u1::text!=t2.u1::text where t1.u1::text like '%000%';
```
И изменяем данные в таблице
```
update test1 set u1=gen_random_uuid() where u1::text like '%';
vacuum;
```
Настройки мастера
```
synchronous_commit='off'
synchronous_standby_names=''
```
Настройки реплики
```
hot_standby_feedback=off
max_standby_streaming_delay='30s'
```
Получаем ошибку через ~30 сек
```
ERROR:  canceling statement due to conflict with recovery
DETAIL:  User query might have needed to see row versions that must be removed.
Time: 32961.169 ms (00:32.961)
```
Включаем синхронный режим и повторяем тест
```
synchronous_commit='local'
synchronous_standby_names='rep1'
```
Также получаем ошибку через ~30 сек
```
ERROR:  canceling statement due to conflict with recovery
DETAIL:  User query might have needed to see row versions that must be removed.
Time: 33618.328 ms (00:33.618)
```
Поменяем параметр на реплике и повторим тест
```
alter system set hot_standby_feedback='on'; select pg_reload_conf();
```
Запрос отработал без ошибки за минуту
```
   count   
-----------
 417900000
(1 row)

Time: 68977.381 ms (01:08.977)
```
Из лога на синхронной реплике видно, что при hot_standby_feedback=on нет ошибки User query might have needed to see row versions that must be removed
```
2024-06-01 10:34:47.804 LOG:  parameter "hot_standby_feedback" changed to "on"
2024-06-01 10:59:10.231 ERROR:  canceling statement due to conflict with recovery
2024-06-01 10:59:10.231 DETAIL:  User was holding a relation lock for too long.

2024-06-01 12:23:04.078 LOG:  parameter "hot_standby_feedback" changed to "off"
2024-06-01 12:43:16.681 ERROR:  canceling statement due to conflict with recovery
2024-06-01 12:43:16.681 DETAIL:  User query might have needed to see row versions that must be removed.

2024-06-01 12:44:47.840 LOG:  parameter "hot_standby_feedback" changed to "on"
2024-06-01 12:45:17.384 ERROR:  canceling statement due to conflict with recovery
2024-06-01 12:45:17.384 DETAIL:  User was holding a relation lock for too long.

2024-06-01 16:08:59.519 LOG:  parameter "hot_standby_feedback" changed to "off"
2024-06-01 16:16:50.202 ERROR:  canceling statement due to conflict with recovery
2024-06-01 16:16:50.202 DETAIL:  User query might have needed to see row versions that must be removed.

2024-06-01 17:14:48.014 LOG:  parameter "hot_standby_feedback" changed to "on"
2024-06-01 17:18:36.283 ERROR:  canceling statement due to conflict with recovery
2024-06-01 17:18:36.283 DETAIL:  User was holding shared buffer pin for too long.

2024-06-01 17:21:29.703 LOG:  parameter "hot_standby_feedback" changed to "off"
2024-06-01 17:21:49.346 ERROR:  canceling statement due to conflict with recovery
2024-06-01 17:21:49.346 DETAIL:  User query might have needed to see row versions that must be removed.
```

# Исследование влияния параметров на накатку валов

Мастер
```
 synchronous_commit           | local
 synchronous_standby_names    | rep1
```
Реплика
```
 hot_standby                  | on
 hot_standby_feedback         | on
 max_standby_streaming_delay  | 30s
```
Во время работы запроса и параллельного обновления таблицы валы приходят и применяются (receive=replay)
```
postgres=# SELECT
  pg_is_in_recovery() AS is_slave,
  pg_last_wal_receive_lsn() AS receive,
  pg_last_wal_replay_lsn() AS replay,
  pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() AS synced,
  (
   EXTRACT(EPOCH FROM now()) -
   EXTRACT(EPOCH FROM pg_last_xact_replay_timestamp())
  )::int AS lag;
 is_slave |  receive   |   replay   | synced | lag 
----------+------------+------------+--------+-----
 t        | 0/74DCA248 | 0/74DCA248 | t      |   5
(1 row)

 is_slave |  receive   |   replay   | synced | lag 
----------+------------+------------+--------+-----
 t        | 0/78BCCF48 | 0/78BCCF48 | t      |   1
(1 row)

```
В асинхронном режиме также приходят и применяются
```
alter system set synchronous_standby_names='';
select pg_reload_conf();
select application_name, client_addr, sync_state from pg_stat_replication;
 application_name | client_addr  | sync_state 
------------------+--------------+------------
 rep1             | 192.168.1.52 | async
```
Установим hot_standby=off и повторим тест
```
postgres=# SELECT
  pg_is_in_recovery() AS is_slave,
  pg_last_wal_receive_lsn() AS receive,
  pg_last_wal_replay_lsn() AS replay,
  pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() AS synced,
  (
   EXTRACT(EPOCH FROM now()) -
   EXTRACT(EPOCH FROM pg_last_xact_replay_timestamp())
  )::int AS lag;
 is_slave |  receive   |   replay   | synced | lag 
----------+------------+------------+--------+-----
 t        | 0/9B562568 | 0/93322510 | f      |  20
(1 row)

 is_slave |  receive   |   replay   | synced | lag 
----------+------------+------------+--------+-----
 t        | 0/9F570B10 | 0/93322510 | f      |  30
(1 row)
```
Теперь валы приходят, но не применяются

Также вернулась ошибка
```
ERROR:  canceling statement due to conflict with recovery
DETAIL:  User query might have needed to see row versions that must be removed.
```
Другая комбинация параметров

Мастер
```
 synchronous_commit         | remote_apply
 synchronous_standby_names  | rep1
```
Реплика
```
 hot_standby_feedback       | on
```
Валы приходят и применяются

При hot_standby_feedback=off в зависимости от значения synchronous_commit и размера буфера валов на мастере возможна полная остановка обновлений на время работы долгого запроса на реплике

Реп1 - при hot_standby_feedback=off не применяет валы
```
postgres=# SELECT
  pg_is_in_recovery() AS is_slave,
  pg_last_wal_receive_lsn() AS receive,
  pg_last_wal_replay_lsn() AS replay,
  pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() AS synced,
  (
   EXTRACT(EPOCH FROM now()) -
   EXTRACT(EPOCH FROM pg_last_xact_replay_timestamp())
  )::int AS lag;
 is_slave |  receive   |   replay   | synced | lag 
----------+------------+------------+--------+-----
 t        | 0/E34C2690 | 0/E31F17D0 | f      |   3
(1 row)

 is_slave |  receive   |   replay   | synced | lag 
----------+------------+------------+--------+-----
 t        | 0/E912EB60 | 0/E31F17D0 | f      |  17
(1 row)
```
Реп2 - каскадная реплика продолжает принимать и применять валы
```
postgres=# SELECT
  pg_is_in_recovery() AS is_slave,
  pg_last_wal_receive_lsn() AS receive,
  pg_last_wal_replay_lsn() AS replay,
  pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() AS synced,
  (
   EXTRACT(EPOCH FROM now()) -
   EXTRACT(EPOCH FROM pg_last_xact_replay_timestamp())
  )::int AS lag;
 is_slave |  receive   |   replay   | synced | lag 
----------+------------+------------+--------+-----
 t        | 0/E34C26C8 | 0/E34C26C8 | t      |   7
(1 row)

 is_slave |  receive   |   replay   | synced | lag 
----------+------------+------------+--------+-----
 t        | 0/E912EB60 | 0/E912EB60 | t      |   0
(1 row)
```
Долгий запрос на каскадной реплике приводит к лагу на ней самой, обновления на мастере не тормозятся, т.к. мастер синхронизируется только со своей репликой
```
здесь без примера запроса
```
Выводы

- при синхронной репликации TPS меньше, чем при асинхронной
- рекомендации включать hot_standby_feedback в случае, когда долгие запросы на реплике тормозят применение валов, подтвержаются
- hot_standby_feedback=on на реплике позволяет сохранить на мастере данные, которые нужны для долгого запроса
- hot_standby_feedback уменьшает лаг

Ссылки

- https://dataegret.com/2015/09/postgresql-hot-standby-feedback-how-it-works/
