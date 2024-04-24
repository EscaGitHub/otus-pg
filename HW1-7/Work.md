## Логический уровень PostgreSQL
Домашнее задание 1 месяц 7 занятие

### Работа с PostgreSQL 15
- Подключаемся к VM
```bash
ssh -i keypair esca@84.201.177.190
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
- Работа с БД **testdb**. **Note:** При работе с таблицей обязательно используем префикс схемы testnm
```bash
postgres=# create database testdb; -- создание БД
CREATE DATABASE
postgres=# \c testdb -- подключение к БД
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm; -- создание схемы
CREATE SCHEMA
testdb=# create table testnm.t1 (c1 integer); -- создание таблицы в схеме
CREATE TABLE 
testdb=# insert into testnm.t1 values (1); -- добавление строки
INSERT 0 1
testdb=# select * from testnm.t1; -- проверка таблицы
 c1
----
  1
(1 row)
```
- Работа с ролью **readonly**
```bash
testdb=# create role readonly; -- создание роли
CREATE ROLE
testdb=# grant usage on schema testnm to readonly; -- выдача прав на схему
GRANT
testdb=# grant connect on database testdb to readonly; -- выдача прав на БД
GRANT
testdb=# grant select on all tables in schema testnm to readonly; -- выдача прав на чтение таблиц в схеме
GRANT
```
- Проверяем, что права на таблицу есть
```bash
testdb=# SELECT grantee, table_schema, table_name, privilege_type
FROM information_schema.table_privileges
WHERE grantee = 'readonly';
 grantee  | table_schema | table_name | privilege_type
----------+--------------+------------+----------------
 readonly | testnm       | t1         | SELECT
(1 row)
```
- Создаем нового пользователя **testread** и добавляем ему роль
```bash
testdb=# create user testread with password 'test123'
testdb-# ;
CREATE ROLE
testdb=# grant readonly to testread;
GRANT ROLE
```
- Заходим под новым пользователем
```bash
esca@otus-pg-vm1:~$ psql -h 127.0.0.1 -U testread -d testdb -W
```
- Проверяем доступ к таблице **t1**, доступ на получение данных есть
```bash
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)
```
- Так же проверяем, где таблица
```bash
testdb=> \dt testnm.*
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 testnm | t1   | table | postgres
(1 row)
```
- Шаги с удалением таблицы из public и создания в нужной схеме не делал. Т.к. сразу создавалось там. 
После пересоздания, добавления новых таблиц, нужно опять выдавать права на все таблицы пользователю 
иначе прав не будет.
- Что бы не перевыдавать права на объекты можно изменить права по умолчанию для роли **readonly** для новых таблиц
```bash
testdb=# alter default privileges in schema testnm grant select on tables to readonly;
ALTER DEFAULT PRIVILEGES
```
- Вызов команды на создание новой таблицы под testread для схемы public не сработал - недостаточно прав
```bash
testdb=> create table t2 (c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t2 (c1 integer);
```
- Права на схемах public и testnm. Таблица t3 - создана из под postgres в схеме public.
```bash
testdb=# \dp public.*
                            Access privileges
 Schema | Name | Type  | Access privileges | Column privileges | Policies
--------+------+-------+-------------------+-------------------+----------
 public | t3   | table |                   |                   |
(1 row)

testdb=# \dp testnm.*
                                Access privileges
 Schema | Name | Type  |     Access privileges     | Column privileges | Policies
--------+------+-------+---------------------------+-------------------+----------
 testnm | t1   | table | postgres=arwdDxt/postgres+|                   |
        |      |       | readonly=r/postgres       |                   |
(1 row)
```