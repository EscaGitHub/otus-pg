## MVCC, vacuum и autovacuum
Домашнее задание 1 месяц 8 занятие

### Работа с PostgreSQL 15

- Подключаемся к VM
```bash
ssh -i keypair esca@51.250.108.195
```
- Проверяем кластер
```bash
sudo -u postgres pg_lsclusters

Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log
```
- Подключаемся как postgres
```bash
sudo -u postgres psql
```
- Создаем БД
```bash
postgres=# create database test_vacuum;
CREATE DATABASE
```
- Заходим пользователем postgres
```bash
esca@otus-pg-vm1:~$ su - postgres
```
- Запускаем тест производительности: pgbench -i [ другие-параметры ] имя_базы
  
https://postgrespro.ru/docs/postgrespro/10/pgbench

Note: Практически во всех случаях, чтобы получить полезные результаты, необходимо передать какие-либо дополнительные параметры. 
Наиболее важные параметры: -c (число клиентов), -t (число транзакций), -T (длительность) и -f (файл со скриптом).

```bash
postgres@otus-pg-vm1:~$ pgbench -i postgres
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 2.13 s (drop tables 0.72 s, create tables 0.38 s, client-side generate 0.31 s, vacuum 0.36 s, primary keys 0.37 s).
```
- Запускаем вторую проверку

-c клиенты
--client=клиенты
Число имитируемых клиентов, то есть число одновременных сеансов базы данных. Значение по умолчанию — 1.

-P сек
--progress=сек
Выводить отчёт о прогрессе через заданное число секунд (сек). 

-T секунды
--time=секунды
Выполнять тест с ограничением по времени (в секундах), а не по числу транзакций для каждого клиента. 
Параметры -t и -T являются взаимоисключающими.
```bash
postgres@otus-pg-vm1:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 305.2 tps, lat 26.103 ms stddev 18.889, 0 failed
progress: 12.0 s, 364.2 tps, lat 21.912 ms stddev 15.602, 0 failed
progress: 18.0 s, 483.0 tps, lat 16.597 ms stddev 11.483, 0 failed
progress: 24.0 s, 282.3 tps, lat 28.351 ms stddev 31.683, 0 failed
progress: 30.0 s, 584.8 tps, lat 13.672 ms stddev 8.512, 0 failed
progress: 36.0 s, 516.2 tps, lat 15.514 ms stddev 12.082, 0 failed
progress: 42.0 s, 488.7 tps, lat 16.368 ms stddev 11.369, 0 failed
progress: 48.0 s, 489.8 tps, lat 16.314 ms stddev 10.378, 0 failed
progress: 54.0 s, 357.2 tps, lat 22.417 ms stddev 23.288, 0 failed
progress: 60.0 s, 625.0 tps, lat 12.803 ms stddev 7.558, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 26986
number of failed transactions: 0 (0.000%)
latency average = 17.785 ms
latency stddev = 15.674 ms
initial connection time = 14.623 ms
tps = 449.756577 (without initial connection time)
```
### Изменение настроек кластера
- Изменяем настройки в файле postgresql.conf, взял из чата telegram, т.к. файл с настройками в материалах не приложен
```text
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 65536kB
min_wal_size = 4GB
max_wal_size = 16GB
```
- [Описание всех настроек на изменение](Config.md)
- Рестартим кластер
```bash
esca@otus-pg-vm1:/$ sudo -u postgres pg_ctlcluster 15 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@15-main
esca@otus-pg-vm1:/$ sudo -u postgres pg_ctlcluster 15 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-main
esca@otus-pg-vm1:/$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log
```
- Проверяем с новыми настройками первый тест
```bash
postgres@otus-pg-vm1:~$ pgbench -i postgres
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.64 s (drop tables 0.69 s, create tables 0.29 s, client-side generate 0.17 s, vacuum 0.11 s, primary keys 0.37 s).
```
- Проверяем с новыми настройками второй тест
```bash
postgres@otus-pg-vm1:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 305.3 tps, lat 26.038 ms stddev 28.337, 0 failed
progress: 12.0 s, 502.8 tps, lat 15.937 ms stddev 15.211, 0 failed
progress: 18.0 s, 525.0 tps, lat 15.227 ms stddev 9.951, 0 failed
progress: 24.0 s, 484.5 tps, lat 16.341 ms stddev 12.584, 0 failed
progress: 30.0 s, 224.8 tps, lat 35.852 ms stddev 41.911, 0 failed
progress: 36.0 s, 438.8 tps, lat 18.271 ms stddev 14.224, 0 failed
progress: 42.0 s, 572.8 tps, lat 13.963 ms stddev 9.107, 0 failed
progress: 48.0 s, 512.7 tps, lat 15.586 ms stddev 11.425, 0 failed
progress: 54.0 s, 499.0 tps, lat 15.935 ms stddev 11.083, 0 failed
progress: 60.0 s, 243.0 tps, lat 33.156 ms stddev 28.358, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 25861
number of failed transactions: 0 (0.000%)
latency average = 18.554 ms
latency stddev = 18.736 ms
initial connection time = 16.081 ms
tps = 430.981111 (without initial connection time)
```
- Изменения
  - до настроек
    ```text
    done in 2.13 s (drop tables 0.72 s, create tables 0.38 s, client-side generate 0.31 s, vacuum 0.36 s, primary keys 0.37 s)
    ```
  - после настроек
    ```text
    done in 1.64 s (drop tables 0.69 s, create tables 0.29 s, client-side generate 0.17 s, vacuum 0.11 s, primary keys 0.37 s).
    ```
    Уменьшилось время генерации таблиц и время выполнения вакуума из-за увеличения пулов разрешенной памяти в настройках кластера.


- Текущие настройки vacuum
```bash
SELECT name, setting, context, short_desc FROM pg_settings WHERE name like 'vacuum%';

               name                |  setting   | context |                                     short_desc

-----------------------------------+------------+---------+-------------------------------------------------------------------------------------
 vacuum_cost_delay                 | 0          | user    | Vacuum cost delay in milliseconds.
 vacuum_cost_limit                 | 200        | user    | Vacuum cost amount available before napping.
 vacuum_cost_page_dirty            | 20         | user    | Vacuum cost for a page dirtied by vacuum.
 vacuum_cost_page_hit              | 1          | user    | Vacuum cost for a page found in the buffer cache.
 vacuum_cost_page_miss             | 2          | user    | Vacuum cost for a page not found in the buffer cache.
 vacuum_defer_cleanup_age          | 0          | sighup  | Number of transactions by which VACUUM and HOT cleanup should be deferred, if any.
 vacuum_failsafe_age               | 1600000000 | user    | Age at which VACUUM should trigger failsafe to avoid a wraparound outage.
 vacuum_freeze_min_age             | 50000000   | user    | Minimum age at which VACUUM should freeze a table row.
 vacuum_freeze_table_age           | 150000000  | user    | Age at which VACUUM should scan whole table to freeze tuples.
 vacuum_multixact_failsafe_age     | 1600000000 | user    | Multixact age at which VACUUM should trigger failsafe to avoid a wraparound outage.
 vacuum_multixact_freeze_min_age   | 5000000    | user    | Minimum age at which VACUUM should freeze a MultiXactId in a table row.
 vacuum_multixact_freeze_table_age | 150000000  | user    | Multixact age at which VACUUM should scan whole table to freeze tuples.
```
- Ставим расширение pgstattuple через psql
```bash
create extension pgstattuple;
CREATE EXTENSION
```
- Смотрим состояние таблицы
```bash
 select * from pgstattuple('pgbench_accounts') \gx
-[ RECORD 1 ]------+---------
table_len          | 13434880   -- размер таблицы в байтах
tuple_count        | 100000     -- количество живых кортежей
tuple_len          | 12100000   -- суммарный размер живых кортежей в байтах
tuple_percent      | 90.06      -- процентное соотношение размера живых кортежей к общему размеру таблицы
dead_tuple_count   | 0          -- количество мертвых кортежей
dead_tuple_len     | 0          -- суммарный размер мертвых кортежей в байтах
dead_tuple_percent | 0          -- процентное соотношение размера мертвых кортежей к общему размеру таблицы
free_space         | 188960     -- объём свободного пространства в таблице в байтах
free_percent       | 1.41       -- процентное соотношение свободного пространства к общему размеру таблицы
```

### Работа с таблицей

- Создаем таблицу и заполняем 1кк строк
```postgresql
testdb=# CREATE TABLE student( id serial, fio text );
CREATE TABLE
testdb=# INSERT INTO student(fio) SELECT (MD5(random()::text)) FROM generate_series(1,1000000);
INSERT 0 1000000
```
- Размер файла
```postgresql
testdb=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 pg_size_pretty
----------------
 65 MB
(1 row)
```
- Смотрим данные автовакуума и мертвых строк до обновления
```postgresql
testdb=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 student |    1000000 |          0 |      0 | 2024-05-01 07:08:19.271283+00
```
- 5 раз обновляем строки и добавить символ
```postgresql
DO $$
DECLARE
    i INTEGER := 0;
BEGIN
    FOR i IN 1..5 LOOP
        UPDATE student SET fio = fio || 'N';
        RAISE NOTICE 'Шаг цикла: %', i;
    END LOOP;
END$$;
```
- Смотрим данные таблицы
```postgresql
testdb=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 student |    1000000 |    5000000 |    499 | 2024-05-01 07:08:19.271283+00
(1 row)
```
- Смотрим когда сработал автовакуум
```postgresql
testdb=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 student |     998661 |          0 |      0 | 2024-05-01 07:22:33.225258+00
(1 row)
```
- Еще 5 раз обновляем таблицу
```postgresql
DO $$
DECLARE
    i INTEGER := 0;
BEGIN
    FOR i IN 1..5 LOOP
        UPDATE student SET fio = fio || 'Y';
        RAISE NOTICE 'Шаг цикла: %', i;
    END LOOP;
END$$;
```
- Смотрим размер файла
```postgresql
testdb=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 pg_size_pretty
----------------
 438 MB
(1 row)
```
- Выключаем автовакуум для таблицы
```postgresql
testdb=# ALTER TABLE student SET (autovacuum_enabled = false);
ALTER TABLE
```
- Обновляем все строки 10 раз
```postgresql
testdb=# DO $$
    DECLARE
        i INTEGER := 0;
    BEGIN
        FOR i IN 1..10 LOOP
                UPDATE student SET fio = fio || 'O';
                RAISE NOTICE 'Шаг цикла: %', i;
            END LOOP;
    END$$;
NOTICE:  Шаг цикла: 1
NOTICE:  Шаг цикла: 2
NOTICE:  Шаг цикла: 3
NOTICE:  Шаг цикла: 4
NOTICE:  Шаг цикла: 5
NOTICE:  Шаг цикла: 6
NOTICE:  Шаг цикла: 7
NOTICE:  Шаг цикла: 8
NOTICE:  Шаг цикла: 9
NOTICE:  Шаг цикла: 10
DO
```
- Смотрим размер таблицы
```postgresql
testdb=# SELECT pg_size_pretty(pg_total_relation_size('student'));
 pg_size_pretty
----------------
 879 MB
(1 row)
```
- И данные по ней
```postgresql
testdb=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float AS "ratio%", last_autovacuum FROM pg_stat_user_tables WHERE relname = 'student';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 student |     990610 |   10000000 |   1009 | 2024-05-01 07:29:33.880631+00
(1 row)
```
- Объяснение размера файла: было 438 MB стало 879MB - размер увеличился в 2 раза, т.к. обновление было 10 раз без включенного
автовакуума и копились скопированные(новые с измененными данными) кортэжи, без удаления старых и остались старые строки с флагом удален.
Часть новых строк влезло в уже "растянутый" файл, а часть пришлось увеличивать.\
Ниже, в разделе дополнительно, можно посмотреть, что процентное соотношение размера живых кортежей к общему размеру 
таблицы очень маленькое, после прохождения автовакуума, т.е. большую часть занимают пустые данные.
### Задание со * - выполнено в рамках запросов обновления строк в таблице выше.
### Дополнительно
- Включили автовакуум обратно
```postgresql
testdb=# ALTER TABLE student SET (autovacuum_enabled = true);
ALTER TABLE
```
- Посмотрим еще состояние таблицы в конце, после отработки автовакуума
```postgresql
testdb=#  select * from pgstattuple('student') \gx
-[ RECORD 1 ]------+----------
table_len          | 921845760
tuple_count        | 1000000
tuple_len          | 81000000
tuple_percent      | 8.79 -- процентное соотношение размера живых кортежей к общему размеру таблицы
dead_tuple_count   | 0
dead_tuple_len     | 0
dead_tuple_percent | 0
free_space         | 826289940
free_percent       | 89.63 -- процентное соотношение свободного пространства к общему размеру таблицы
```