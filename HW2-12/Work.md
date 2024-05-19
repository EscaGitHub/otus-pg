## Резервное копирование и восстановление
Домашнее задание 2 месяц 12 занятие

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

### Работа с таблицей
- Создаем БД
```postgresql
create database testdb;
CREATE DATABASE
\c testdb
```
- Переходим и создаем таблицу student со 100 записями
```postgresql
testdb=# CREATE TABLE student( id serial, fio text );
CREATE TABLE
testdb=# INSERT INTO student(fio) SELECT (MD5(random()::text)) FROM generate_series(1,100);
INSERT 0 100
postgres=# SELECT * FROM student limit 5;
 id |               fio
----+----------------------------------
  1 | 53b4c55c3ab6a00c4c86070d7b487cfe
  2 | f3c31dc51062e720d96bfea34ad4d081
  3 | 97922e254221f340183c1ba00341e8da
  4 | dff03c5575f66d99a78fcb589523605f
  5 | 79a7bccfa6da6b8bc50565aeb743d4e1
(5 rows)
```
### Бэкап COPY
- Создаем папку для backup в примонтированной области
```bash
cd /mnt
sudo mkdir backup
sudo chown -R postgres:postgres /mnt/backup/
```
- Создаем backup твблицы в примонтированную папку
```postgresql
testdb=# \copy student to '/mnt/backup/backup_copy.sql';
COPY 100
```
- Создаем вторую таблицу в которую будем восстанавливать
```postgresql
testdb=# CREATE TABLE studentbackup( id serial, fio text );
CREATE TABLE
testdb=# select * from studentbackup;
 id | fio
----+-----
(0 rows)
```
- Восстанавливаем данные в нее из файла backup
```postgresql
testdb=# \copy studentBackup from '/mnt/backup/backup_copy.sql';
COPY 100
testdb=# testdb=# select * from studentbackup limit 5;
id |               fio
----+----------------------------------
  1 | 83c476fa3e207351c7080d943a70eb76
  2 | eca0bcc7d97be2c49dc71cc33c0a350c
  3 | a9fc850036cd46a2236888c02628eea4
  4 | 1471d961f6e299dba002419e7b51511c
  5 | 93470e45b1779cb98f9ff9e25b4bb91e
(5 rows)
```

### Бэкап pg_dump
- От пользователя postgres создаем бэкап БД testdb в которой 2 таблицы
```bash
postgres@otus-pg-vm1:~$ pg_dump -d testdb --create -Fc > /mnt/backup/backup_dump.gz
postgres@otus-pg-vm1:~$ ls /mnt/backup -l
total 12
-rw-rw-r-- 1 postgres postgres 3592 May 19 08:16 backup_copy.sql
-rw-rw-r-- 1 postgres postgres 8141 May 19 08:44 backup_dump.gz
```
- Описание параметров для формата вывода копии
```text
-F format
--format=format
Указывает формат вывода копии. format может принимать следующие значения:

p
plain
Сформировать текстовый SQL-скрипт. Это поведение по умолчанию.

c
custom
Выгрузить данные в специальном архивном формате, пригодном для дальнейшего использования утилитой pg_restore. 
Наряду с форматом directory является наиболее гибким форматом, позволяющим вручную выбирать и сортировать 
восстанавливаемые объекты. Вывод в этом формате по умолчанию сжимается.

d
directory
Выгрузить данные в формате каталога. Этот формат пригоден для дальнейшего использования утилитой pg_restore. 
При этом будет создан каталог, в котором для каждой таблицы и большого объекта будут созданы отдельные файлы, 
а также файл оглавления в машинно-читаемом формате, понятном для pg_restore. С полученной резервной копией можно 
работать штатными средствами Unix, например, несжатую копию можно сжать посредством gzip. Этот формат по умолчанию 
сжимается, а также поддерживает работу в несколько потоков.
```
- Если нужно выгрузить именно две таблицы, то можно воспользоваться флагом -t
```bash
postgres@otus-pg-vm1:~$ pg_dump -d testdb -t student -t studentbackup --create -Fc > /mnt/backup/tables_backup_dump.gz
postgres@otus-pg-vm1:~$ ls /mnt/backup -l
total 20
-rw-rw-r-- 1 postgres postgres 3592 May 19 08:16 backup_copy.sql
-rw-rw-r-- 1 postgres postgres 8141 May 19 08:44 backup_dump.gz
-rw-rw-r-- 1 postgres postgres 8141 May 19 08:54 tables_backup_dump.gz
```
- Создаем новую БД
```postgresql
postgres=# create database testdbbackup;
CREATE DATABASE
```
- Восстанавливаем из backup только вторую таблицу studentbackup
  - -d - в какую БД
  - -t - какую таблицу
```bash
postgres@otus-pg-vm1:~$ pg_restore -d testdbbackup -t studentbackup /mnt/backup/tables_backup_dump.gz
```
- Проверяем, что все появилось
```postgresql
postgres=# \c testdbbackup
You are now connected to database "testdbbackup" as user "postgres".
testdbbackup=# \dt
List of relations
Schema |     Name      | Type  |  Owner
--------+---------------+-------+----------
public | studentbackup | table | postgres
(1 row)

testdbbackup=# select * from studentbackup limit 5;
id |               fio
----+----------------------------------
1 | 83c476fa3e207351c7080d943a70eb76
2 | eca0bcc7d97be2c49dc71cc33c0a350c
3 | a9fc850036cd46a2236888c02628eea4
4 | 1471d961f6e299dba002419e7b51511c
5 | 93470e45b1779cb98f9ff9e25b4bb91e
(5 rows)
```

