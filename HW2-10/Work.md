## Блокировки
Домашнее задание 2 месяц 10 занятие

### Работа с PostgreSQL 15

- Подключаемся к VM
```bash
ssh -i keypair esca@158.160.88.138
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

### Настройка блокировок
- Включаем логирование блокировок
```postgresql
postgres=# show log_lock_waits;
 log_lock_waits
----------------
 off
(1 row)

postgres=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```
- Включаем логи блокировок от 200 мс
```postgresql
postgres=# SHOW deadlock_timeout;
 deadlock_timeout
------------------
 1s
(1 row)

postgres=# ALTER SYSTEM SET deadlock_timeout = 200;
ALTER SYSTEM
postgres=#  SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
 
postgres=#  SHOW deadlock_timeout;
deadlock_timeout
------------------
 200ms
(1 row)
```
- Создаем и заполняем таблицу
```postgresql
CREATE TABLE student( id serial, fio text );
INSERT INTO student(fio) SELECT (MD5(random()::text)) FROM generate_series(1,100);
testdb=# select * from student limit 5;
id    |               fio
---------+----------------------------------
 1000001 | c05c3e3b266169f3c14d03da491dba9e
 1000002 | b7ed21179624a2c13ae3e4d3e6e7395a
 1000003 | d085c433ab3f8ac70e3b08326756edb0
 1000004 | 9d0fdf1f3e3821fa96fa989c5300bd8f
 1000005 | 5479b89ab8fc415b2613c64953a10b65
(5 rows)
```
- 1 сеанс - Изменяем строку в транзакции
```postgresql
BEGIN;
UPDATE student SET fio = '1' WHERE id = 1000001;
```
- 2 сеанс - Изменяем строку в транзакции
```postgresql
BEGIN;
UPDATE student SET fio = '2' WHERE id = 1000001;
```
- Транзакции заблокировались, смотрим логи
```bash
cat /var/log/postgresql/postgresql-15-main.log;
 
2024-05-12 06:55:42.841 UTC [15224] postgres@testdb DETAIL:  Process holding the lock: 15097. Wait queue: 15224.
2024-05-12 06:55:42.841 UTC [15224] postgres@testdb CONTEXT:  while updating tuple (0,1) in relation "student"
2024-05-12 06:55:42.841 UTC [15224] postgres@testdb STATEMENT:  UPDATE student SET fio = '2' WHERE id = 1000001;
2024-05-12 06:56:14.274 UTC [15224] postgres@testdb LOG:  process 15224 acquired ShareLock on transaction 455003 after 31633.483 ms
2024-05-12 06:56:14.274 UTC [15224] postgres@testdb CONTEXT:  while updating tuple (0,1) in relation "student"
2024-05-12 06:56:14.274 UTC [15224] postgres@testdb STATEMENT:  UPDATE student SET fio = '2' WHERE id = 1000001;
```
### PG_LOCKS
- Запускаем транзакции на обновление одной строки в 3 сеансах
```postgresql
BEGIN;
UPDATE student SET fio = '1' WHERE id = 1000001;
```
- Проверяем блокировки - первый сеанс (1629) заблокировал второй сеанс (1625), а второй сеанс (1625) заблокировал третий (1630).
Tuple блокировки означают, что блокировка идет на кортеже(строке) в таблице. false в granted означает, что другой запрос
владеет блокировкой и текущий запрос не может продолжаться.
```postgresql
testdb=*# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for
          FROM pg_locks WHERE relation = 'student'::regclass;
locktype |       mode       | granted | pid  | wait_for
----------+------------------+---------+------+----------
 relation | RowExclusiveLock | t       | 1625 | {1629}
 relation | RowExclusiveLock | t       | 1629 | {}
 relation | RowExclusiveLock | t       | 1630 | {1625}
 tuple    | ExclusiveLock    | f       | 1630 | {1625}
 tuple    | ExclusiveLock    | t       | 1625 | {1629}
(5 rows)
```
- Смотрим pg_locks для текущего процесса - у нас есть блокировка на строке на таблице student
```postgresql
testdb=*# SELECT pg_backend_pid();
pg_backend_pid
----------------
          1629
(1 row)

testdb=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
                    FROM pg_locks WHERE pid = 1629;
   locktype    | relation | virtxid |  xid   |       mode       | granted
---------------+----------+---------+--------+------------------+---------
 relation      | pg_locks |         |        | AccessShareLock  | t
 relation      | student  |         |        | RowExclusiveLock | t
 virtualxid    |          | 4/4     |        | ExclusiveLock    | t
 transactionid |          |         | 455012 | ExclusiveLock    | t
(4 rows)
```
### Взаимоблокировка трех транзакций
- 1 сеанс
```postgresql
BEGIN;
UPDATE student SET fio = '1' WHERE id = 1000001;
```
- 2 сеанс
```postgresql
BEGIN;
UPDATE student SET fio = '2' WHERE id = 1000002;
```
- 3 сеанс
```postgresql
BEGIN;
UPDATE student SET fio = '3' WHERE id = 1000003;
```
- 1 сеанс обновляем строку из 2 сеанса
```postgresql
UPDATE student SET fio = '1' WHERE id = 1000002;
```
- 2 сеанс обновляем строку из 3 сеанса
```postgresql
UPDATE student SET fio = '2' WHERE id = 1000003;
```
- 3 сеанс обновляем строку из 1 сеанса - зацикливаем
```postgresql
UPDATE student SET fio = '3' WHERE id = 1000001;
```
- После вызова последней команды видим сообщение
```postgresql
testdb=*# UPDATE student SET fio = '3' WHERE id = 1000001;
ERROR:  deadlock detected
DETAIL:  Process 2227 waits for ShareLock on transaction 455015; blocked by process 2229.
Process 2229 waits for ShareLock on transaction 455016; blocked by process 2228.
Process 2228 waits for ShareLock on transaction 455017; blocked by process 2227.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,103) in relation "student"
```
- Id процессов ``` SELECT pg_backend_pid(); ```:
  - 3 сеанс - 2227 процесс
  - 2 сеанс - 2228 процесс
  - 1 сеанс - 2229 процесс
- Лог взаимоблокировки из файла лога
```bash
2024-05-12 09:59:06.611 UTC [2229] postgres@testdb LOG:  process 2229 still waiting for ShareLock on transaction 455016 after 200.223 ms
2024-05-12 09:59:06.611 UTC [2229] postgres@testdb DETAIL:  Process holding the lock: 2228. Wait queue: 2229.
2024-05-12 09:59:06.611 UTC [2229] postgres@testdb CONTEXT:  while updating tuple (0,2) in relation "student"
2024-05-12 09:59:06.611 UTC [2229] postgres@testdb STATEMENT:  UPDATE student SET fio = '1' WHERE id = 1000002;
2024-05-12 09:59:16.804 UTC [2228] postgres@testdb LOG:  process 2228 still waiting for ShareLock on transaction 455017 after 200.107 ms
2024-05-12 09:59:16.804 UTC [2228] postgres@testdb DETAIL:  Process holding the lock: 2227. Wait queue: 2228.
2024-05-12 09:59:16.804 UTC [2228] postgres@testdb CONTEXT:  while updating tuple (0,3) in relation "student"
2024-05-12 09:59:16.804 UTC [2228] postgres@testdb STATEMENT:  UPDATE student SET fio = '2' WHERE id = 1000003;
2024-05-12 09:59:23.591 UTC [2227] postgres@testdb LOG:  process 2227 detected deadlock while waiting for ShareLock on transaction 455015 after 200.085 ms
2024-05-12 09:59:23.591 UTC [2227] postgres@testdb DETAIL:  Process holding the lock: 2229. Wait queue: .
2024-05-12 09:59:23.591 UTC [2227] postgres@testdb CONTEXT:  while updating tuple (0,103) in relation "student"
2024-05-12 09:59:23.591 UTC [2227] postgres@testdb STATEMENT:  UPDATE student SET fio = '3' WHERE id = 1000001;
2024-05-12 09:59:23.591 UTC [2227] postgres@testdb ERROR:  deadlock detected
2024-05-12 09:59:23.591 UTC [2227] postgres@testdb DETAIL:  Process 2227 waits for ShareLock on transaction 455015; blocked by process 2229.
        Process 2229 waits for ShareLock on transaction 455016; blocked by process 2228.
        Process 2228 waits for ShareLock on transaction 455017; blocked by process 2227.
        Process 2227: UPDATE student SET fio = '3' WHERE id = 1000001;
        Process 2229: UPDATE student SET fio = '1' WHERE id = 1000002;
        Process 2228: UPDATE student SET fio = '2' WHERE id = 1000003;
2024-05-12 09:59:23.591 UTC [2227] postgres@testdb HINT:  See server log for query details.
2024-05-12 09:59:23.591 UTC [2227] postgres@testdb CONTEXT:  while updating tuple (0,103) in relation "student"
2024-05-12 09:59:23.591 UTC [2227] postgres@testdb STATEMENT:  UPDATE student SET fio = '3' WHERE id = 1000001;
2024-05-12 09:59:23.591 UTC [2228] postgres@testdb LOG:  process 2228 acquired ShareLock on transaction 455017 after 6986.807 ms
2024-05-12 09:59:23.591 UTC [2228] postgres@testdb CONTEXT:  while updating tuple (0,3) in relation "student"
2024-05-12 09:59:23.591 UTC [2228] postgres@testdb STATEMENT:  UPDATE student SET fio = '2' WHERE id = 1000003;
```
### Блокировка UPDATE без WHERE
- Вопрос:
```text
Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), 
заблокировать друг друга?
```
- Ответ:\
Да могут, если во время обновления всей таблицы они попадут на обновление одной и той же строки. 
Т.к. блокировки во время такого обновления идут на уровне строки.
#### Пример блокировки Update без Where * 
- Очищаем таблицу student
- Добавляем только одну строку
```postgresql
testdb=# insert into student values (1, '111');
INSERT 0 1
```
- 1 сеанс вызываем обновление таблицы в транзакции
```postgresql
testdb=# BEGIN;
BEGIN
testdb=*# update student set fio = 2;
UPDATE 1
```
- 2 сеанс также пытаемся обновить таблицу без where и зависаем в блокировке
```postgresql
testdb=# begin;
BEGIN
testdb=*# update student set fio = 3;
```
- Блокировки из запроса - процесс 2771 ожидает процесс 2837
```postgresql
testdb=*# SELECT locktype, mode, granted, pid, pg_blocking_pids(pid) AS wait_for
FROM pg_locks WHERE relation = 'student'::regclass;
locktype |       mode       | granted | pid  | wait_for
----------+------------------+---------+------+----------
relation | RowExclusiveLock | t       | 2837 | {}
relation | RowExclusiveLock | t       | 2771 | {2837}
tuple    | ExclusiveLock    | t       | 2771 | {2837}
(3 rows)
```
- Отменяем первую транзакцию, и вторая после этого выполняется
```postgresql
testdb=*# rollback;
ROLLBACK
```
- Лог блокировки из файла
```bash
2024-05-12 10:29:00.764 UTC [2771] postgres@testdb LOG:  process 2771 still waiting for ShareLock on transaction 455025 after 200.096 ms
2024-05-12 10:29:00.764 UTC [2771] postgres@testdb DETAIL:  Process holding the lock: 2837. Wait queue: 2771.
2024-05-12 10:29:00.764 UTC [2771] postgres@testdb CONTEXT:  while updating tuple (0,101) in relation "student"
2024-05-12 10:29:00.764 UTC [2771] postgres@testdb STATEMENT:  update student set fio = 3;
2024-05-12 10:29:25.325 UTC [2771] postgres@testdb LOG:  process 2771 acquired ShareLock on transaction 455025 after 24760.913 ms
2024-05-12 10:29:25.325 UTC [2771] postgres@testdb CONTEXT:  while updating tuple (0,101) in relation "student"
2024-05-12 10:29:25.325 UTC [2771] postgres@testdb STATEMENT:  update student set fio = 3;
```

