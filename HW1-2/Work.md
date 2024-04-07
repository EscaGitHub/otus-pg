## Работа с уровнями изоляции транзакции в PostgreSQL
Домашнее задание 1 месяц 2 занятие

### Создание машины в WSL
- Проверяем какие есть дистрибутивы `wsl --list --online`
- Ставим Ubuntu `wsl --install -d Ubuntu`

### Установка PostgreSQL
https://www.postgresql.org/download/linux/ubuntu/
- Добавляем репозиторий: 
`sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'`
- Добавляем ключ:
`wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -`
- Обновляем пакеты:
`sudo apt-get update`
- Устанавливаем 14 PostgreSQL:
`sudo apt-get -y install postgresql-14`
- Проверяем версию:
`psql --version`
```console
psql (PostgreSQL) 14.11 (Ubuntu 14.11-1.pgdg22.04+1)
```
 - Запускаем сервис: `sudo service postgresql start`
 - Проверить статус `sudo service postgresql status`
```console
14/main (port 5432): online
```
### Работа с PostgreSQL
- Запуск psql `sudo -u postgres psql`
- Выключить auto commit `\set AUTOCOMMIT OFF`
- Проверить `\echo :AUTOCOMMIT`
- Добавить БД otus `create database otus;`
- Перейти в БД otus `\c otus`
- Посмотреть все доступные БД `\l`
- Посмотреть объекты в БД `\d`
- Создаем таблицу `create table persons(id serial, first_name text, second_name text);`
- Заполнили таблицу `insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov');`
- Очистка консоли psql `\! clear`
- Посмотреть уровень изоляции `show transaction isolation level;`
```console
 read committed
```
#### Первое добавление - read committed
- Добавляем новую строку в таблицу в первой сессии: ` insert into persons(first_name, second_name) values('sergey', 'sergeev');`
- Делаем выборку из таблицы во второй сессии: `select * from persons;` - новой записи нет
```console
otus=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
- Сделали commit транзакции в первой сессии
- Во второй сессии сделали выборку из таблицы - новая запись появилась, т.к. стоит уровень изоляции read committed, 
то новые записи мы видим только после коммитов транзакций.
```console
otus=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
#### Второе добавление - repeatable read

- Устанавливаем уровень изоляции для первой сессии: `set transaction isolation level repeatable read;`
- Добавляем запись в первой сессии в таблицу: `insert into persons(first_name, second_name) values('sveta', 'svetova');`
- Начинаем работу во второй сессии: `begin;`
- Устанавливаем уровень изоляции во второй сессии: `set transaction isolation level repeatable read;`
- Сделали во второй сессии выборку из таблицы - новой записи нет
```console
otus=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
- commit в первой сессии
- Делаем выборку во второй сессии - новой записи все еще нет
- Делаем commit во второй сессии
- Делаем выборку во второй сессии - новая запись появилась - т.к. закончилась транзакция с уровнем изоляции 
repeatable read, то теперь видим данные уже не из ее snapshot, а как в дефолтной транзакции.
```console
otus=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```