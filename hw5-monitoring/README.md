# Системы мониторинга на примере PPEM

Стенд

- Debian 12

- CPU 2, RAM 4, HDD 32

## Установка сервера

Для репозитория рекомендуется любая редакция СУБД Postgres Pro, для тестов будем использовать PostgreSQL 15 из Debian
```
wget https://repo.postgrespro.ru/ppem/ppem/keys/pgpro-repo-add.sh

sh pgpro-repo-add.sh

apt update
apt upgrade
apt install pgpro-manager
apt install postgresql-15
```

## Настройка сервера

Такой pg_hba.conf не рекомендуется в производственной эксплуатации
```
-
#local   all             postgres                               peer
+
# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
host    all             all             127.0.0.1/32            trust
```

Инициализация репозитория
```
# init-pgpro-manager-repo --conf /etc/pgpro-manager.conf
repo user: pgpro_manager
Error: cannot connect admdb: connection to server on socket "/tmp/.s.PGSQL.5432" failed: No such file or directory
        Is the server running locally and accepting connections on that socket?
```

Найдем сокет в postgresql.conf и укажем в pgpro-manager.conf
```
unix_socket_directories = '/var/run/postgresql'

host = /var/run/postgresql
```
Снова ошибка
```
# init-pgpro-manager-repo --conf /etc/pgpro-manager.conf
repo user: pgpro_manager
Create user: pgpro_manager
+ user pgpro_manager created
Create database: pgpro_manager_repo owner pgpro_manager
+ database created
Upload sql from file: /usr/share/pgpro-manager/sql/init_repo.sql
Traceback (most recent call last):
  File "/usr/bin/init-pgpro-manager-repo", line 33, in <module>
    sys.exit(load_entry_point('ee-manager==1.4.0', 'console_scripts', 'init-pgpro-manager-repo')())
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3/dist-packages/ee_manager/lib/init_repo.py", line 424, in main
    insert_sql_file(repoconn, schema_file)
  File "/usr/lib/python3/dist-packages/ee_manager/lib/init_repo.py", line 283, in insert_sql_file
    cur.execute(sql)
UnicodeEncodeError: 'ascii' codec can't encode characters in position 4588-4593: ordinal not in range(128)
```
Посмотрим кодировку
```
postgres=# \l
                                                    List of databases
        Name        |     Owner     | Encoding  | Collate | Ctype | ICU Locale | Locale Provider |   Access privileges   
--------------------+---------------+-----------+---------+-------+------------+-----------------+-----------------------
 pgpro_manager_repo | pgpro_manager | SQL_ASCII | C       | C     |            | libc            | 
 postgres           | postgres      | SQL_ASCII | C       | C     |            | libc            | 
 template0          | postgres      | SQL_ASCII | C       | C     |            | libc            | =c/postgres          +
                    |               |           |         |       |            |                 | postgres=CTc/postgres
 template1          | postgres      | SQL_ASCII | C       | C     |            | libc            | =c/postgres          +
                    |               |           |         |       |            |                 | postgres=CTc/postgres

update pg_database set encoding = pg_char_to_encoding('UTF8') where datname = 'template0';
update pg_database set encoding = pg_char_to_encoding('UTF8') where datname = 'template1';
update pg_database set encoding = pg_char_to_encoding('UTF8') where datname = 'postgres';
drop database pgpro_manager_repo;

postgres=# \l
                                                   List of databases
        Name        |     Owner     | Encoding | Collate | Ctype | ICU Locale | Locale Provider |   Access privileges   
--------------------+---------------+----------+---------+-------+------------+-----------------+-----------------------
 postgres           | postgres      | UTF8     | C       | C     |            | libc            | 
 template0          | postgres      | UTF8     | C       | C     |            | libc            | =c/postgres          +
                    |               |          |         |       |            |                 | postgres=CTc/postgres
 template1          | postgres      | UTF8     | C       | C     |            | libc            | =c/postgres          +
                    |               |          |         |       |            |                 | postgres=CTc/postgres
```
Репозиторий инициализирован
```
# init-pgpro-manager-repo --conf /etc/pgpro-manager.conf
repo user: pgpro_manager
Create user: pgpro_manager
* user pgpro_manager already exists
Create database: pgpro_manager_repo owner pgpro_manager
+ database created
Upload sql from file: /usr/share/pgpro-manager/sql/init_repo.sql
+ sql  loaded
Add information about repository instance
Set migration level to: 1405
+ migration level set

systemctl restart pgpro-manager
```
Web-интерфейс PPEM http://ppem:8877

Логин и пароль admin/admin

## Установка и настройка агента на целевом сервере

```
wget https://repo.postgrespro.ru/ppem/ppem/keys/pgpro-repo-add.sh

sh pgpro-repo-add.sh

apt -y update
apt -y upgrade
apt -y install pgpro-manager-agent
```
На сервере PPEM зайти в "Настройки-Настройки агентов-Добавить агент" и добавить агент

После добавления скопировать ключ и добавить адрес сервера PPEM в настройки агента на целевом сервере /etc/pgpro-manager-agent.conf

Затем перезапустить службу pgpro-manager-agent на целевом сервере

Через меню "Экземпляр-Добавить экземпляр" добавить экземпляр с обнаружением, нужно указать пароль пользователя postgres

## Некоторые возможности интерфейса

Отображается показатель Bloat

![](img/img1.png)

DDL и параметры индекса

![](img/img2.png)

Запуск вакуума с параметрами

![](img/img3.png)

Редактирование последовательности

![](img/img4.png)

Файлы таблицы

![](img/img5.png)

## pg_stat_statements

Для отслеживания статистики выполнения запросов нужно загрузить модуль-расширение

Расширение имеет ряд настраиваемых параметров через postgresql.conf
```
alter system set shared_preload_libraries='pg_stat_statements';
restart
```
Пример использования
```
postgres=# SELECT pg_stat_statements_reset();

postgres=# select count(*) from test1 t1 join test2 t2 on t1.u1::text!=t2.u1::text where t1.u1::text like '%000%';
   count   
-----------
 418500000
(1 row)

postgres=# update test1 set u1=gen_random_uuid() where u1::text like '%';
UPDATE 300000

postgres=# SELECT query, calls, total_exec_time, rows, 100.0 * shared_blks_hit /
               nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent, wal_bytes
          FROM pg_stat_statements ORDER BY 3 DESC LIMIT 5;\G
                                               query                                               | calls |  total_exec_time   |  rows  |     hit_percent      | wal_bytes 
---------------------------------------------------------------------------------------------------+-------+--------------------+--------+----------------------+-----------
 select count(*) from test1 t1 join test2 t2 on t1.u1::text!=t2.u1::text where t1.u1::text like $1 |     1 |        71240.32012 |      1 | 100.0000000000000000 |         0
 update test1 set u1=gen_random_uuid() where u1::text like $1                                      |     1 |        1765.123279 | 300000 |  99.9995780501698348 |  94913759
```

## Table bloat или раздувание таблиц

С помощью такого запроса можно оценить степень заполненности датафайлов в пользовательских тейблспейсах

Результат не совпадает с данными из PPEM, но в целом задает направление для дальнейшего исследования

```
/* WARNING: executed with a non-superuser role, the query inspect only tables and materialized view (9.3+) you are granted to read.
* This query is compatible with PostgreSQL 9.0 and more
*/
SELECT current_database(), schemaname, tblname, bs*tblpages AS real_size,
  (tblpages-est_tblpages)*bs AS extra_size,
  CASE WHEN tblpages > 0 AND tblpages - est_tblpages > 0
    THEN 100 * (tblpages - est_tblpages)/tblpages::float
    ELSE 0
  END AS extra_pct, fillfactor,
  CASE WHEN tblpages - est_tblpages_ff > 0
    THEN (tblpages-est_tblpages_ff)*bs
    ELSE 0
  END AS bloat_size,
  CASE WHEN tblpages > 0 AND tblpages - est_tblpages_ff > 0
    THEN 100 * (tblpages - est_tblpages_ff)/tblpages::float
    ELSE 0
  END AS bloat_pct, is_na
  -- , tpl_hdr_size, tpl_data_size, (pst).free_percent + (pst).dead_tuple_percent AS real_frag -- (DEBUG INFO)
FROM (
  SELECT ceil( reltuples / ( (bs-page_hdr)/tpl_size ) ) + ceil( toasttuples / 4 ) AS est_tblpages,
    ceil( reltuples / ( (bs-page_hdr)*fillfactor/(tpl_size*100) ) ) + ceil( toasttuples / 4 ) AS est_tblpages_ff,
    tblpages, fillfactor, bs, tblid, schemaname, tblname, heappages, toastpages, is_na
    -- , tpl_hdr_size, tpl_data_size, pgstattuple(tblid) AS pst -- (DEBUG INFO)
  FROM (
    SELECT
      ( 4 + tpl_hdr_size + tpl_data_size + (2*ma)
        - CASE WHEN tpl_hdr_size%ma = 0 THEN ma ELSE tpl_hdr_size%ma END
        - CASE WHEN ceil(tpl_data_size)::int%ma = 0 THEN ma ELSE ceil(tpl_data_size)::int%ma END
      ) AS tpl_size, bs - page_hdr AS size_per_block, (heappages + toastpages) AS tblpages, heappages,
      toastpages, reltuples, toasttuples, bs, page_hdr, tblid, schemaname, tblname, fillfactor, is_na
      -- , tpl_hdr_size, tpl_data_size
    FROM (
      SELECT
        tbl.oid AS tblid, ns.nspname AS schemaname, tbl.relname AS tblname, tbl.reltuples,
        tbl.relpages AS heappages, coalesce(toast.relpages, 0) AS toastpages,
        coalesce(toast.reltuples, 0) AS toasttuples,
        coalesce(substring(
          array_to_string(tbl.reloptions, ' ')
          FROM 'fillfactor=([0-9]+)')::smallint, 100) AS fillfactor,
        current_setting('block_size')::numeric AS bs,
        CASE WHEN version()~'mingw32' OR version()~'64-bit|x86_64|ppc64|ia64|amd64' THEN 8 ELSE 4 END AS ma,
        24 AS page_hdr,
        23 + CASE WHEN MAX(coalesce(s.null_frac,0)) > 0 THEN ( 7 + count(s.attname) ) / 8 ELSE 0::int END
           + CASE WHEN bool_or(att.attname = 'oid' and att.attnum < 0) THEN 4 ELSE 0 END AS tpl_hdr_size,
        sum( (1-coalesce(s.null_frac, 0)) * coalesce(s.avg_width, 0) ) AS tpl_data_size,
        bool_or(att.atttypid = 'pg_catalog.name'::regtype)
          OR sum(CASE WHEN att.attnum > 0 THEN 1 ELSE 0 END) <> count(s.attname) AS is_na
      FROM pg_attribute AS att
        JOIN pg_class AS tbl ON att.attrelid = tbl.oid
        JOIN pg_namespace AS ns ON ns.oid = tbl.relnamespace
        LEFT JOIN pg_stats AS s ON s.schemaname=ns.nspname
          AND s.tablename = tbl.relname AND s.inherited=false AND s.attname=att.attname
        LEFT JOIN pg_class AS toast ON tbl.reltoastrelid = toast.oid
      WHERE NOT att.attisdropped
        AND tbl.relkind in ('r','m')
      GROUP BY 1,2,3,4,5,6,7,8,9,10
      ORDER BY 2,3
    ) AS s
  ) AS s2
) AS s3
WHERE schemaname not like 'pg_%' and schemaname != 'information_schema'
-- WHERE NOT is_na
--   AND tblpages*((pst).free_percent + (pst).dead_tuple_percent)::float4/100 >= 1
ORDER BY schemaname, tblname;

 current_database | schemaname |     tblname      | real_size | extra_size |     extra_pct      | fillfactor | bloat_size |     bloat_pct      | is_na 
------------------+------------+------------------+-----------+------------+--------------------+------------+------------+--------------------+-------
 postgres         | public     | pgbench_accounts |  15204352 |    1957888 | 12.877155172413794 |        100 |    1957888 | 12.877155172413794 | f
 postgres         | public     | pgbench_branches |     24576 |      16384 |  66.66666666666667 |        100 |      16384 |  66.66666666666667 | f
 postgres         | public     | pgbench_history  |     90112 |       8192 |  9.090909090909092 |        100 |       8192 |  9.090909090909092 | f
 postgres         | public     | pgbench_tellers  |     40960 |      32768 |                 80 |        100 |      32768 |                 80 | f
 postgres         | public     | test1            | 262152192 |  229736448 |  87.63476141370582 |        100 |  229736448 |  87.63476141370582 | f
 postgres         | public     | test2            |  32768000 |     270336 |              0.825 |        100 |     270336 |              0.825 | f
```
Скрипт со stackoverflow для пересоздания раздутых таблиц уменьшил размер таблицы test1 с 250 до 30 Мб
```
postgres=# CREATE OR REPLACE FUNCTION admin.repack_table(text)
RETURNS text
AS $$
DECLARE SQL text;
BEGIN
    SELECT
      'CREATE TEMP TABLE t1 (LIKE '||$1||');'||chr(10)||
      'INSERT INTO t1 SELECT * FROM '||$1||';'||chr(10)||
      'TRUNCATE TABLE '||$1||';'||chr(10)||
      'INSERT INTO '||$1||' SELECT * FROM t1;'||chr(10)||
      'DROP TABLE t1;'||chr(10)||
      'ANALYZE '||$1||';'
    INTO SQL;
    EXECUTE SQL;
    RETURN $1;
END;
$$ LANGUAGE plpgsql;

-- repack all tables in certain schema (with an optional threshold for N of dead tuples)
CREATE OR REPLACE FUNCTION admin.repack_schema(text,int default 5000)
RETURNS table (table_name text)
AS $$
DECLARE SQL text;
BEGIN
RETURN QUERY (
    with schema as (select $1)
    select admin.repack_table(t.table_schema||'.'||t.table_name)
    from information_schema.tables t
    where t.table_schema=(select * from schema)
    and t.table_name in (
        select relname
        from pg_stat_all_tables
        where schemaname=(select * from schema)
        and n_dead_tup>$2
        and n_live_tup<1000000 -- avoid repacking too large tables
    )
);
END;
$$ LANGUAGE plpgsql;

postgres=# select public.repack_schema('public');
 repack_schema 
---------------
(0 rows)

 current_database | schemaname |     tblname      | real_size | extra_size |     extra_pct      | fillfactor | bloat_size |     bloat_pct      | is_na 
------------------+------------+------------------+-----------+------------+--------------------+------------+------------+--------------------+-------
 postgres         | public     | pgbench_accounts |  15204352 |    1957888 | 12.877155172413794 |        100 |    1957888 | 12.877155172413794 | f
 postgres         | public     | pgbench_branches |     24576 |      16384 |  66.66666666666667 |        100 |      16384 |  66.66666666666667 | f
 postgres         | public     | pgbench_history  |     90112 |       8192 |  9.090909090909092 |        100 |       8192 |  9.090909090909092 | f
 postgres         | public     | pgbench_tellers  |     40960 |      32768 |                 80 |        100 |      32768 |                 80 | f
 postgres         | public     | test1            |  32768000 |     270336 |              0.825 |        100 |     270336 |              0.825 | f
 postgres         | public     | test2            |  32768000 |     270336 |              0.825 |        100 |     270336 |              0.825 | f
```

Ссылки

- https://postgrespro.ru/docs/postgresql/15/pgstatstatements
- https://wiki.postgresql.org/wiki/Show_database_bloat
- https://habr.com/ru/articles/169939/
- https://stackoverflow.com/questions/59456694/table-bloat-on-postgres