# Подключение и настройка балансировщика нагрузки и мониторинга PostgreSQL

## Введение
Создание соединения с базой данных PostgreSQL для выполнения запросов обходится довольно дорого. Накладные расходы 
составляют около 1.3MB - 2MB памяти на одно соединение. Так же каждый раз нужны ресурсы для созданий [новых процессов
на сервере БД](https://www.postgresql.org/docs/current/connect-estab.html). Каждый клиент подключается к новому процессу,
которыми управляет postmaster. Всякий раз, когда он обнаруживает запрос на соединение, он запускает новый внутренний 
процесс. При большом количестве клиентских соединений БД создает множество процессов и управляет разделяемыми 
структурами данных. Из-за этого может возникнуть нехватка вычислительных ресурсов, которая сказывается на производительности.

Чтобы решить проблему нехватки ресурсов, перед кластером PostgreSQL часто размещают менеджеры подключений (connection 
pooler). Менеджеры подключений управляют соединениями, позволяя подключиться к БД большому числу клиентов без деградации
производительности. Между менеджером подключений и СУБД поддерживается относительно небольшое количество соединений, 
которые можно переиспользовать. После отключения клиента соединение возвращается в пул и может быть повторно использовано
тем же самым или новым клиентом.
<img src="./pic/pools.png" alt="drawing" width="1051"/>

## Виды

### [Pgpool-II](https://www.pgpool.net/)

- Если нам требуется N параллельных соединений, это создает N дочерних процессов. По умолчанию существует 32 дочерних процесса.
- Определяет один процесс для каждого дочернего процесса. Мы не можем контролировать, к какому дочернему процессу подключается клиент.
Клиент получает выгоду от пулового соединения только в том случае, если он подключается к дочернему процессу, который 
ранее обслуживал соединение для этой комбинации БД+пользователь.
- Поддерживает только сессионный режим работы.
- Обеспечивает высокую доступность - автоматическое переключение на реплики.
- Поддерживает автоматическую балансировку нагрузки — он даже достаточно интеллектуален, чтобы перенаправлять запросы на
чтение на реплики и запись на мастер.
- Позволяет ограничивать только общее количество соединений.

### [Pgbouncer](https://www.pgbouncer.org/)

- Использует только один процесс - легковесный.
- Создает новый пул в комбинации пользователя и БД. Поэтому он общий для всех клиентов.
- Поддерживает три вида режима работы:
  - сессионный - клиент получает постоянное подключение к серверу на все время своей сессии. Когда клиент завершает сессию, 
  подключение возвращается в пул. Этот метод является наиболее безопасным и используется по умолчанию.
  - транзакционный - клиент получает подключение только на время выполнения транзакции, после завершения которой 
  подключение возвращается в пул.
  - операторов - самый агрессивный метод. Клиентские подключения возвращается в пул после каждого выполненного запроса. 
  При использовании этого метода нельзя выполнять транзакции, содержащие несколько операторов, поскольку после каждого 
  оператора подключение закрывается.
- Нет балансировки нагрузки.
- Позволяет ограничивать количество подключений для каждого пула, базы данных, пользователя или клиента.
- Сквозная аутентификация

### [Odyssey](https://yandex.ru/dev/odyssey/) - Yandex

- Многопоточная обработка.
- Продвинутый transaction pooling.
- Правильная передача ошибок клиенту.
- Удобное логирование и отладка.
- Детальная настройка пулов.
- Совместимость с консолью Pgbouncer.

## Установка инфраструктуры
### Prometheus
Ставим Prometheus в docker с пробросом файла настроек и данных:
```bash
docker run --name prometheus -p 9090:9090 -v D:\docker\volumes\prometheus.yml:/etc/prometheus/prometheus.yml -v D:\docker\volumes\prometheus-data:/prometheus prom/prometheus
```
Новый запуск контейнера:
```bash
docker start -ai prometheus
```
Доступ
```bash
http://localhost:9090
```

### Grafana
Устанавливаем docker
```bash
docker run -d -p 3000:3000 --name=grafana grafana/grafana-enterprise
```
Доступ (admin admin)
```bash
http://localhost:3000/login
```
Подключаем prometheus из докера: 
+ Home -> Connections -> Data sources -> Выбираем prometheus
+ Т.к. Prometheus в docker необходимо получить его ip
```bash
# Берем CONTAINER ID prometheus:
docker ps -a 
# Порлучаем ip:
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' CONTAINER ID
```
+ В строке 'Prometheus server URL' прописываем полученный адрес до prometheus:
```bash
http://172.17.0.3:9090 
```
+ Импортируем dashboard для метрик prometheus, метрики пошли:
<img src="./pic/metrics_prom.png" alt="drawing" width="1051"/>

## Установка и настройка PgBouncer
- Подключаемся к VM
```bash
ssh -i keypair esca@89.169.168.245
```
- Ставим пакет
```bash
sudo apt-get -y install pgbouncer
```
- Редактируем /etc/pgbouncer/pgbouncer.ini
```ini
;; database name = connect string
;;
;; connect string params:
;;   dbname= host= port= user= password= auth_user=
;;   client_encoding= datestyle= timezone=
;;   pool_size= reserve_pool= max_db_connections=
;;   pool_mode= connect_query= application_name=
[databases]

* = host=localhost port=5432

;; IP address or * which means all IPs
listen_addr = *
```
- Добавляем пользователя для доступа в файл userlist.txt в этом же каталоге, пароль в md5 формате (ностроено в pgbouncer.ini)
```bash
# Такой формат для md5: 
echo -n "md5"; echo -n "PASSLOGIN" | md5sum | awk '{print $1}'
sudo nano userlist.txt
```
- Но у нас включено в postgres sha-256, поэтому будем через него. Достаем из postgres пароль в sha для нужного пользователя
```postgresql
SELECT usename,passwd FROM pg_shadow;
```
- Меняем тип аутентификации в pgbouncer.ini - по умолчанию md5, но нам не подходит
```ini
auth_type=scram-sha-256
```
- И прописываем в файл настроек userlist.txt
```ini
"postgres" "SCRAM-SHA-256$4096:QJD4TIe7uUKGk3l23mBWqA==$Iu6sRqpA7iVYWi5R1NfoP8TY/HEuBYFoatDkq11nehY=:eho0GVCw5n12qtabf9/MXNxvkYzz5xMLSiFcXrKRmW0="
```
- Запускаем - одновременно включаем (enable) и запускаем (start) службы systemd. При использовании этой команды служба \
будет автоматически запускаться при старте системы и будет запущена в текущем сеансе без необходимости перезапуска системы.
```bash
sudo systemctl enable --now pgbouncer

sudo systemctl status pgbouncer

● pgbouncer.service - connection pooler for PostgreSQL
     Loaded: loaded (/lib/systemd/system/pgbouncer.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2024-08-04 09:13:12 UTC; 25min ago
       Docs: man:pgbouncer(1)
             https://www.pgbouncer.org/
   Main PID: 19599 (pgbouncer)
     Status: "stats: 0 xacts/s, 0 queries/s, 0 client parses/s, 0 server parses/s, 0 binds/s, in 0 B/s, out 0 B/s, xact 0 μs, query >
      Tasks: 2 (limit: 4556)
     Memory: 1.4M
        CPU: 218ms
     CGroup: /system.slice/pgbouncer.service
             └─19599 /usr/sbin/pgbouncer /etc/pgbouncer/pgbouncer.ini
```
- Проверяем доступ 6432 - порт балансера, вводим обычный пароль в plain-text формате для пользователя 
```bash
psql -h localhost -p 6432 -U postgres postgres
Password for user postgres:
psql (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
Type "help" for help.

postgres=>
```
- Пробуем нагрузку 
```bash
pgbench -p 6432 -c 5 -j 16 -T 10 -U postgres -h localhost postgres

pgbench (15.6 (Ubuntu 15.6-1.pgdg22.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 5
number of threads: 5
maximum number of tries: 1
duration: 10 s
number of transactions actually processed: 2496
number of failed transactions: 0 (0.000%)
latency average = 19.946 ms
initial connection time = 66.578 ms
tps = 250.679854 (without initial connection time)
```
## Установка и настройка pgbouncer_exporter

Экспортер метрик pbBouncer для Prometheus. Написан на Go. Метрики будут по-умолчанию экспортироваться в 9127/metrics.
+ Идем в настройки pgbouncer.ini b и прописываем доступ для пользователя экспортера, в users он так же должен быть добавлен
```yaml
;; comma-separated list of users who are just allowed to use SHOW command
stats_users = postgres
```
+ Рестартим pgbouncer
```bash
sudo systemctl reload pgbouncer.service
```
+ Устанавливаем через docker образ с флагами, где прописываем доступ к БД pgbouncer и пробрасываем порт метрик
```bash
docker run -d -p 9127:9127 prometheuscommunity/pgbouncer-exporter --pgBouncer.connectionString="postgres://<user>:<password>@<postgres db ip>/pgbouncer?sslmode=disable"
```
Были проблемы со строкой подключения и докер не хотел запускаться.
+ Полезное для разворачивания docker
```bash
# Список container
docker ps -a 
# Выполнение комманды внутри docker
docker exec -it 6ccf202636fb command
# Удалить все контейнеры, созданные от image
docker rm $(docker ps -a --filter ancestor=prometheuscommunity/pgbouncer-exporter:latest -q) 
# Запуск ядра docker
sudo service docker stop
sudo service docker start
```
+ Проверяем доступность получаемых [метрик](metrcis_under_loading.txt), внутри VM и потом на внешнем компьютере c prometheus
```bash
curl --location 'http://89.169.161.153:9127/metrics'

# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 2.4223e-05
go_gc_duration_seconds{quantile="0.25"} 6.4102e-05
go_gc_duration_seconds{quantile="0.5"} 7.8818e-05
go_gc_duration_seconds{quantile="0.75"} 0.000110635
go_gc_duration_seconds{quantile="1"} 0.000575917
go_gc_duration_seconds_sum 0.002880287
go_gc_duration_seconds_count 25 
```
+ Идем в настройки promethеus.yml добавляем scrape
```yaml
  - job_name: 'postgres-exporter'
    metrics_path: '/metrics'
    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s
    static_configs:
      - targets: ['89.169.161.153:9127']
```
+ Рестартим prometheus для применения настроек, проверяем что новый target появился и работает
<img src="./pic/prometheus_pgbouncer.png" alt="drawing" width="1051"/>

+ Втягиваем dashboard в Grafana
<img src="./pic/pgbouncer_firstrun.png" alt="drawing" width="1051"/>

+ Нагружаем тестом и смотрим, что бы метрики пошли
```bash
pgbench -p 6432 -c 5 -j 16 -T 60 -U postgres -h localhost postgres
```
+ [Дорисовываем](PgBouncer-1723381883497.json) и изменяем метрики в Grafana
<img src="./pic/grafana_metrics1.png" alt="drawing" width="1051"/>
<img src="./pic/grafana_metrics2.png" alt="drawing" width="1051"/>

## Источники
- https://habr.com/ru/articles/499404/ - Управление нагрузкой на PostgreSQL, когда одного сервера уже мало. Андрей Сальников
- https://yandex.cloud/ru/docs/managed-postgresql/concepts/pooling?utm_referrer=https%3A%2F%2Fwww.google.com%2F - Управление соединениями
- https://stackoverflow.blog/2020/10/14/improve-database-performance-with-connection-pooling/ - Improve database 
performance with connection pooling. October 2020.
- https://scalegrid.io/blog/postgresql-connection-pooling-part-4-pgbouncer-vs-pgpool/ - PostgreSQL® Connection Pooling: 
Part 4 – PgBouncer vs. Pgpool-II. Jul 29, 2020.
- https://postgrespro.ru/docs/postgrespro/15/pgbouncer - pgbouncer
- https://hub.docker.com/r/prometheuscommunity/pgbouncer-exporter - prometheuscommunity/pgbouncer-exporter
- https://tembo.io/blog/postgres-connection-poolers - Benchmarking PostgreSQL connection poolers: PgBouncer, PgCat and Supavisor.
Feb 13, 2024
- https://grafana.com/grafana/dashboards/14022-pgbouncer/ - PgBouncer Grafana dashboard
- https://github.com/prometheus-community/pgbouncer_exporter - PgBouncer Prometheus exporter