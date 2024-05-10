## Журналы
Домашнее задание 2 месяц 9 занятие

### Работа с PostgreSQL 15

- Подключаемся к VM
```bash
ssh -i keypair esca@51.250.102.237
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
### Checkpoint
- Настраиваем выполнение контрольной точки раз в 30 секунд
```bash
postgres=# ALTER SYSTEM SET checkpoint_timeout = '30s';
ALTER SYSTEM
```
- Перезагружаем кластер
```bash
 sudo -u postgres pg_ctlcluster 15 main restart
```
- Текущая настройка checkpoint
```bash
postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
(1 row)
```
- Проверяем текущий LSN (Log Sequence Number) WAL
```bash
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 1/1AFCE060
(1 row)
```
- Запускаем pgbench на 10 минут
```bash
 su - postgres
 
postgres@otus-pg-vm1:~$ pgbench -P 60 -T 600 -U postgres postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 447.1 tps, lat 2.236 ms stddev 2.763, 0 failed
progress: 120.0 s, 391.1 tps, lat 2.556 ms stddev 2.870, 0 failed
progress: 180.0 s, 452.5 tps, lat 2.210 ms stddev 2.765, 0 failed
progress: 240.0 s, 414.9 tps, lat 2.409 ms stddev 3.047, 0 failed
progress: 300.0 s, 388.3 tps, lat 2.575 ms stddev 3.034, 0 failed
progress: 360.0 s, 380.9 tps, lat 2.625 ms stddev 3.003, 0 failed
progress: 420.0 s, 372.2 tps, lat 2.686 ms stddev 2.914, 0 failed
progress: 480.0 s, 329.8 tps, lat 3.031 ms stddev 3.592, 0 failed
progress: 540.0 s, 409.1 tps, lat 2.444 ms stddev 2.789, 0 failed
progress: 600.0 s, 439.3 tps, lat 2.275 ms stddev 2.490, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 241518
number of failed transactions: 0 (0.000%)
latency average = 2.484 ms
latency stddev = 2.927 ms
initial connection time = 3.082 ms
tps = 402.531335 (without initial connection time)
```
- Смотрим новый LSN
```bash
postgres=# SELECT pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn
---------------------------
 1/326E8140
(1 row)

postgres=#
```
- Оцениваем размер
```bash
postgres=# SELECT ('1/326E8140'::pg_lsn - '1/1AFCE060'::pg_lsn)/1024/1024 || ' MB';
        ?column?
-------------------------
 375.1017761230468750 MB
(1 row)
```
- Объем в среднем на одну контрольную точку - 375 / 20 = 18 MB
- За статистикой идем в log файл под пользователем postgres и берем промежуток за наш тест
```bash
 grep checkpoint /var/log/postgresql/postgresql-15-main.log
 
2024-05-10 06:26:13.093 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:26:40.067 UTC [1822] LOG:  checkpoint complete: wrote 1957 buffers (1.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.890 s, sync=0.027 s, total=26.975 s; sync files=22, longest=0.014 s, average=0.002 s; distance=17095 kB, estimate=17095 kB
2024-05-10 06:26:43.069 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:27:10.099 UTC [1822] LOG:  checkpoint complete: wrote 1803 buffers (1.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.982 s, sync=0.012 s, total=27.030 s; sync files=13, longest=0.008 s, average=0.001 s; distance=19081 kB, estimate=19081 kB
2024-05-10 06:27:13.100 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:27:40.058 UTC [1822] LOG:  checkpoint complete: wrote 1866 buffers (1.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.871 s, sync=0.033 s, total=26.958 s; sync files=6, longest=0.014 s, average=0.006 s; distance=18001 kB, estimate=18973 kB
2024-05-10 06:27:43.061 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:28:10.127 UTC [1822] LOG:  checkpoint complete: wrote 1920 buffers (1.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.981 s, sync=0.033 s, total=27.067 s; sync files=16, longest=0.012 s, average=0.003 s; distance=17849 kB, estimate=18860 kB
2024-05-10 06:28:13.128 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:28:40.114 UTC [1822] LOG:  checkpoint complete: wrote 1948 buffers (1.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.870 s, sync=0.059 s, total=26.986 s; sync files=7, longest=0.030 s, average=0.009 s; distance=19553 kB, estimate=19553 kB
2024-05-10 06:28:43.117 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:29:10.108 UTC [1822] LOG:  checkpoint complete: wrote 1992 buffers (1.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.906 s, sync=0.037 s, total=26.991 s; sync files=17, longest=0.013 s, average=0.003 s; distance=20512 kB, estimate=20512 kB
2024-05-10 06:29:13.111 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:29:40.138 UTC [1822] LOG:  checkpoint complete: wrote 1932 buffers (1.5%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.883 s, sync=0.077 s, total=27.027 s; sync files=7, longest=0.047 s, average=0.011 s; distance=19816 kB, estimate=20442 kB
2024-05-10 06:29:43.141 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:30:10.090 UTC [1822] LOG:  checkpoint complete: wrote 2004 buffers (1.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.874 s, sync=0.015 s, total=26.950 s; sync files=16, longest=0.010 s, average=0.001 s; distance=19988 kB, estimate=20397 kB
2024-05-10 06:30:13.093 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:30:40.351 UTC [1822] LOG:  checkpoint complete: wrote 1830 buffers (1.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.973 s, sync=0.212 s, total=27.258 s; sync files=8, longest=0.098 s, average=0.027 s; distance=18056 kB, estimate=20163 kB
2024-05-10 06:30:43.354 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:31:10.098 UTC [1822] LOG:  checkpoint complete: wrote 1920 buffers (1.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.673 s, sync=0.024 s, total=26.744 s; sync files=15, longest=0.013 s, average=0.002 s; distance=18078 kB, estimate=19954 kB
2024-05-10 06:31:13.101 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:31:40.070 UTC [1822] LOG:  checkpoint complete: wrote 1789 buffers (1.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.897 s, sync=0.033 s, total=26.969 s; sync files=7, longest=0.014 s, average=0.005 s; distance=17708 kB, estimate=19730 kB
2024-05-10 06:31:43.071 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:32:10.133 UTC [1822] LOG:  checkpoint complete: wrote 1901 buffers (1.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.970 s, sync=0.017 s, total=27.062 s; sync files=16, longest=0.011 s, average=0.001 s; distance=18147 kB, estimate=19572 kB
2024-05-10 06:32:13.136 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:32:40.120 UTC [1822] LOG:  checkpoint complete: wrote 1854 buffers (1.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.880 s, sync=0.016 s, total=26.985 s; sync files=6, longest=0.007 s, average=0.003 s; distance=17820 kB, estimate=19396 kB
2024-05-10 06:32:43.121 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:33:10.075 UTC [1822] LOG:  checkpoint complete: wrote 1878 buffers (1.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.874 s, sync=0.017 s, total=26.954 s; sync files=14, longest=0.016 s, average=0.002 s; distance=17693 kB, estimate=19226 kB
2024-05-10 06:33:13.078 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:33:40.158 UTC [1822] LOG:  checkpoint complete: wrote 1771 buffers (1.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.964 s, sync=0.040 s, total=27.081 s; sync files=6, longest=0.022 s, average=0.007 s; distance=17875 kB, estimate=19091 kB
2024-05-10 06:33:43.161 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:34:10.114 UTC [1822] LOG:  checkpoint complete: wrote 1878 buffers (1.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.887 s, sync=0.018 s, total=26.954 s; sync files=15, longest=0.011 s, average=0.002 s; distance=17325 kB, estimate=18914 kB
2024-05-10 06:34:13.117 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:34:40.112 UTC [1822] LOG:  checkpoint complete: wrote 1807 buffers (1.4%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.880 s, sync=0.052 s, total=26.995 s; sync files=6, longest=0.035 s, average=0.009 s; distance=17709 kB, estimate=18794 kB
2024-05-10 06:34:43.115 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:35:10.065 UTC [1822] LOG:  checkpoint complete: wrote 1892 buffers (1.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.883 s, sync=0.009 s, total=26.950 s; sync files=14, longest=0.009 s, average=0.001 s; distance=18626 kB, estimate=18777 kB
2024-05-10 06:35:13.068 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:35:40.101 UTC [1822] LOG:  checkpoint complete: wrote 1755 buffers (1.3%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.974 s, sync=0.019 s, total=27.034 s; sync files=7, longest=0.011 s, average=0.003 s; distance=17783 kB, estimate=18678 kB
2024-05-10 06:35:43.104 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:36:10.030 UTC [1822] LOG:  checkpoint complete: wrote 2044 buffers (1.6%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.877 s, sync=0.020 s, total=26.927 s; sync files=14, longest=0.020 s, average=0.002 s; distance=18867 kB, estimate=18867 kB
2024-05-10 06:36:43.061 UTC [1822] LOG:  checkpoint starting: time
2024-05-10 06:37:10.059 UTC [1822] LOG:  checkpoint complete: wrote 1786 buffers (1.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.972 s, sync=0.012 s, total=26.998 s; sync files=16, longest=0.011 s, average=0.001 s; distance=16512 kB, estimate=18631 kB
```
- Отличается только последняя запись - это может быть из-за уже низкой активности после окончания теста pgbench.
### Асинхронный режим
- Проверяем значение параметра синхронизации
```bash
postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 on
(1 row)
```
Запускаем первый тест
```bash
postgres@otus-pg-vm1:~$ pgbench -T 30 -U postgres postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 30 s
number of transactions actually processed: 10575
number of failed transactions: 0 (0.000%)
latency average = 2.837 ms
initial connection time = 3.103 ms
tps = 352.522467 (without initial connection time)
```
- Выключаем и проверяем
```bash
postgres=# ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 off
(1 row)

postgres=#
```
- Запускаем тест еще раз
```bash
postgres@otus-pg-vm1:~$ pgbench -T 30 -U postgres postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
maximum number of tries: 1
duration: 30 s
number of transactions actually processed: 66653
number of failed transactions: 0 (0.000%)
latency average = 0.450 ms
initial connection time = 3.165 ms
tps = 2221.979310 (without initial connection time)
```
- Сравнение tps
```bash
tps = 352.522467 (without initial connection time)
tps = 2221.979310 (without initial connection time)
```
Параметр synchronous_commit в PostgreSQL отвечает за поведение базы данных по отношению к подтверждению транзакций. 
Значение этого параметра определяет, будет ли транзакция ждать записи WAL (журнала предзаписи) на диск перед тем, 
как сообщить клиенту о её успешном завершении (COMMIT).
- Когда synchronous_commit включен (on), PostgreSQL гарантирует, что после получения подтверждения о транзакции (COMMIT) 
данные сохранены на диске и не будут потеряны при сбое системы.
- При выключении synchronous_commit (off), сервер сообщает об успешном выполнении COMMIT ещё до того, как связанные с ним 
данные записаны на диск. Это может повысить производительность при большом количестве транзакций, но в случае сбоя 
не гарантируется сохранность данных транзакций, зафиксированных после последней записи WAL на диск.

### Кластер с включенными контрольными суммами
- Создаем второй кластер с проверкой checksum на другом порту --data-checksums
```bash
esca@otus-pg-vm1:/var/log$ sudo pg_createcluster 15 second --port 5433 -- --data-checksums
Creating new PostgreSQL cluster 15/second ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/second --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/15/second ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster Port Status Owner    Data directory                Log file
15  second  5433 down   postgres /var/lib/postgresql/15/second /var/log/postgresql/postgresql-15-second.log
```
- Для удаления кластера можно использовать
```bash
sudo pg_dropcluster 15 second
```
- Запускаем
```bash
sudo -u postgres pg_ctlcluster 15 second start
```
- Проверяем
```bash
esca@otus-pg-vm1:/var/log$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
15  main    5432 online postgres /mnt/data/15/main             /var/log/postgresql/postgresql-15-main.log
15  second  5433 online postgres /var/lib/postgresql/15/second /var/log/postgresql/postgresql-15-second.log
```
- Подключаемся к новому кластеру
```bash
sudo -u postgres psql -p 5433
```
- Создаем и заполняем табилцу
```postgresql
t=# CREATE TABLE student( id serial, fio text );
CREATE TABLE
t=# INSERT INTO student(fio) SELECT (MD5(random()::text)) FROM generate_series(1,100);
INSERT 0 100
t=# select * from student;
```
- Находим oid таблицы
```postgresql
postgres=# SELECT oid, relfilenode FROM pg_class WHERE relname = 'student';
  oid  | relfilenode
-------+-------------
 16396 |       16396
(1 row)
```
- Каталог с файлами
```bash
postgres=# show data_directory;
        data_directory
-------------------------------
 /var/lib/postgresql/15/second
(1 row)
```
- Останавливаем кластер
```bash
esca@otus-pg-vm1:/var/log$ sudo -u postgres pg_ctlcluster 15 second stop
esca@otus-pg-vm1:/var/log$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
15  main    5432 online postgres /mnt/data/15/main             /var/log/postgresql/postgresql-15-main.log
15  second  5433 down   postgres /var/lib/postgresql/15/second /var/log/postgresql/postgresql-15-second.log
```
- Находим файл с таким oid
```bash
postgres@otus-pg-vm1:~/15/second/base$ find . -type f -name '*16396*'
./16391/16396
```
- Редактируем файл - добавим в конце 32442fd
- Стартуем кластер, заходим, пытаемся вызвать SELECT - падает ошибка проверки сумм, т.к. мы изменили файл
```bash
t=# select * from student;
WARNING:  page verification failed, calculated checksum 44905 but expected 49792
ERROR:  invalid page in block 0 of relation base/16391/16396
```
- Включаем ignore_checksum_failure (boolean) - При обнаружении ошибок контрольных сумм при чтении Postgres Pro обычно
сообщает об ошибке и прерывает текущую транзакцию. Если параметр ignore_checksum_failure включён, система игнорирует 
проблему (но всё же предупреждает о ней) и продолжает обработку.
```postgresql
 alter system set ignore_checksum_failure = on;
 SELECT pg_reload_conf();
 ```
- Вызываем опять SELECT на таблице - данные выводятся
```postgresql
t=# select * from student limit 5;
 id |               fio
----+----------------------------------
  1 | bb3d39e528866d82807fa87eba647f6d
  2 | f94ca45f2657b2fcac0850513ad5bbcc
  3 | eee465542ac5b1a574e72eab39b21b7f
  4 | ed17dd1963482e47621b2253b13a4ae3
  5 | a0e58d53d87153adf56d4cd86360965e
(5 rows)
```

