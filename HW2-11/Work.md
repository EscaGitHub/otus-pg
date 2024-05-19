## Настройка PostgreSQL
Домашнее задание 2 месяц 11 занятие

### Работа с PostgreSQL 15

- Подключаемся к VM
```bash
ssh -i keypair esca@84.201.153.140
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

### Нагрузка и конфигурация кластера
- Первый запуск с текущими настройками
```bash
postgres@otus-pg-vm1:~$ pgbench -c 50 -j 2 -P 10 -T 60 postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 370.5 tps, lat 13.461 ms stddev 19.446, 0 failed
progress: 20.0 s, 353.4 tps, lat 14.097 ms stddev 17.887, 0 failed
progress: 30.0 s, 236.6 tps, lat 21.042 ms stddev 32.715, 0 failed
progress: 40.0 s, 292.3 tps, lat 17.137 ms stddev 22.272, 0 failed
progress: 50.0 s, 277.6 tps, lat 17.968 ms stddev 23.171, 0 failed
progress: 60.0 s, 244.4 tps, lat 20.548 ms stddev 27.206, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 5
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 17753
number of failed transactions: 0 (0.000%)
latency average = 16.899 ms
latency stddev = 23.704 ms
initial connection time = 7.747 ms
tps = 295.833850 (without initial connection time)
```
- Описание параметров pgbench:
    - -c Клиенты. Число имитируемых клиентов, то есть число одновременных сеансов базы данных. Значение по умолчанию — 1.
    - -j Потоки. Число рабочих потоков в pgbench. Использовать нескольких потоков может быть полезно на многопроцессорных компьютерах. Клиенты распределяются по доступным потокам равномерно, насколько это возможно. Значение по умолчанию — 1.
    - -P Секунды. Выводить отчёт о прогрессе через заданное число секунд (сек). Выдаваемый отчёт включает время, прошедшее с момента запуска, скорость (в TPS) с момента предыдущего отчёта, а также среднее время ожидания транзакций и стандартное отклонение. В режиме ограничения скорости (-R) время ожидания вычисляется относительно назначенного времени запуска транзакции, а не фактического времени её начала, так что оно включает и среднее время отставания от графика.
    - -T Cекунды. Выполнять тест с ограничением по времени (в секундах), а не по числу транзакций для каждого клиента. Параметры -t и -T являются взаимоисключающими.
- Смотрим включена ли возможность подключения настроек и в какой директории
```bash
include_dir = 'conf.d'                  # include files ending in '.conf' from
```
- Переходим туда под пользователем postgres
```bash
postgres@otus-pg-vm1:~$ cd /etc/postgresql/15/main/conf.d/
```
- Идем на сайт pgtune вводим конфигурацию VM для генерации оптимальных настроек и получаем:
```bash
# DB Version: 15
# OS Type: linux
# DB Type: mixed
# Total Memory (RAM): 4 GB
# CPUs num: 2
# Connections num: 100
# Data Storage: hdd

max_connections = 100
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 2621kB
huge_pages = off
min_wal_size = 1GB
max_wal_size = 4GB
```
- Создаем в директории conf.d новый файл с настройками - pgtune.conf
- Рестартим кластер для применения настроек
```bash
esca@otus-pg-vm1:~$ sudo pg_ctlcluster 15 main restart
```
- Смотрим текущие настройки для параметра shared_buffers
```postgresql
postgres=# select * from pg_file_settings where name='shared_buffers';
sourcefile                 | sourceline | seqno |      name      | setting | applied | error
--------------------------------------------+------------+-------+----------------+---------+---------+-------
 /etc/postgresql/15/main/postgresql.conf    |        127 |    11 | shared_buffers | 128MB   | f       |
 /etc/postgresql/15/main/conf.d/pgtune.conf |         10 |    34 | shared_buffers | 1GB     | t       |
(2 rows)
```
- Запукскаем pgbench для новых настроек
```bash
postgres@otus-pg-vm1:~$  pgbench -c 50 -j 2 -P 10 -T 60 postgres
pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
progress: 10.0 s, 260.1 tps, lat 187.419 ms stddev 209.862, 0 failed
progress: 20.0 s, 344.0 tps, lat 145.681 ms stddev 143.981, 0 failed
progress: 30.0 s, 384.4 tps, lat 130.059 ms stddev 108.135, 0 failed
progress: 40.0 s, 269.4 tps, lat 186.232 ms stddev 176.875, 0 failed
progress: 50.0 s, 418.8 tps, lat 119.243 ms stddev 104.802, 0 failed
progress: 60.0 s, 335.1 tps, lat 149.230 ms stddev 130.773, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 20168
number of failed transactions: 0 (0.000%)
latency average = 148.817 ms
latency stddev = 146.318 ms
initial connection time = 75.282 ms
tps = 335.547077 (without initial connection time)
```
- Сравнение результатов
```bash
Было:
latency average = 16.899 ms
latency stddev = 23.704 ms
initial connection time = 7.747 ms
tps = 295.833850 (without initial connection time)

Стало:
latency average = 148.817 ms
latency stddev = 146.318 ms
initial connection time = 75.282 ms
tps = 335.547077 (without initial connection time)
```
- Описание параметров:
  - latency average - указывает на среднее время ожидания выполнения запроса в миллисекундах
  - latency stddev - указывает на стандартное отклонение времени ожидания выполнения запроса в миллисекундах
  - initial connection time - указывает на среднее время установления начального соединения с базой данных в миллисекундах
  - tps - указывает на количество транзакций в секунду (Transactions Per Second), которое было обработано во время тестирования, 
  за исключением времени установления начального соединения
- Вывод - увеличились время ожидания выполнения запроса, отклонение времени ожидания и среднее время установления соединения.
Выросло количество транзакций в секунд, что даст нам большую производительность по операциям.
- [Описание всех настроек на изменение предложенных pgtune](Config.md)