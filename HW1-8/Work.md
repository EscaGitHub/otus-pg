## MVCC, vacuum и autovacuum
Домашнее задание 1 месяц 8 занятие

### Работа с PostgreSQL 15

- Подключаемся к VM
```bash
ssh -i keypair esca@158.160.14.241
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
100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 1.50 s (drop tables 0.22 s, create tables 0.03 s, client-side generate 0.92 s, vacuum 0.07 s, primary keys 0.26 s).
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
progress: 6.0 s, 597.0 tps, lat 13.348 ms stddev 10.631, 0 failed
progress: 12.0 s, 614.5 tps, lat 13.015 ms stddev 10.206, 0 failed
progress: 18.0 s, 553.0 tps, lat 14.466 ms stddev 9.171, 0 failed
progress: 24.0 s, 577.5 tps, lat 13.855 ms stddev 8.623, 0 failed
progress: 30.0 s, 240.0 tps, lat 30.417 ms stddev 35.324, 0 failed
progress: 36.0 s, 511.8 tps, lat 16.979 ms stddev 38.633, 0 failed
progress: 42.0 s, 432.5 tps, lat 18.490 ms stddev 12.648, 0 failed
progress: 48.0 s, 430.7 tps, lat 18.599 ms stddev 14.149, 0 failed
progress: 54.0 s, 546.5 tps, lat 14.631 ms stddev 11.530, 0 failed
progress: 60.0 s, 303.7 tps, lat 26.351 ms stddev 33.092, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 28851
number of failed transactions: 0 (0.000%)
latency average = 16.636 ms
latency stddev = 20.119 ms
initial connection time = 15.198 ms
tps = 480.670245 (without initial connection time)
```
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


- Ставим расширение через psql
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

- Скрипт на 10 повторений обновления таблицы
```postgresql
DO $$
DECLARE
    i INTEGER := 0;
BEGIN
    FOR i IN 1..10 LOOP
        UPDATE t3 SET c1 = 1;
        RAISE NOTICE 'Шаг цикла: %', i;
    END LOOP;
END$$;
```