# Нагрузочное тестирование и тюнинг PostgreSQL
### Цели:
* сделать нагрузочное тестирование PostgreSQL
* настроить параметры PostgreSQL для достижения максимальной производительности

# Решение домашнего задания
Исходные данные:
* Виртуальная машина: Ubuntu 24.04.2 LTS, 16 Gb RAM, 4 CPU, 10 Gb SSD
* PostgreSQL: 16.9

## Настройка базовой производительности
В сервисе https://pgtune.leopard.in.ua/ выставить параметры системы:
* DB version: 16
* OS Type: Linux
* DB Type: Web application
* Total Memory: 16 Gb
* Number of CPUs: 4
* Number of Connections: 30
* Data Storage: SSD storage

Система выдает следующие рекомендации по настройках кластера:
```
# DB Version: 16
# OS Type: linux
# DB Type: web
# Total Memory (RAM): 16 GB
# CPUs num: 4
# Connections num: 30
# Data Storage: ssd

max_connections = 30
shared_buffers = 4GB
effective_cache_size = 12GB
maintenance_work_mem = 1GB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 123361kB
huge_pages = off
min_wal_size = 1GB
max_wal_size = 4GB
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_workers = 4
max_parallel_maintenance_workers = 2
```
Настроить выставить их в [db.yml](../deploy/vm/group_vars/db.yml).
Примерить изменения на сервере командой:
```shell
> ansible-playbook playbooks/install_db.yml -l pg-master
```
После перезапуска сервера подключиться к нему и убедиться, что изменения применены с помощью запроса:
```sql
SELECT name, setting, boot_val, source
FROM pg_settings
WHERE setting <> boot_val;
```
## Тестирование производительности
Зайти на сервер по SSH и выполнить нагрузочное тестирование с помощью pgbench:
```shell
> sudo su - postgres
> pgbench -c 25 -j 8 -T 300 persons
```
Что запустит нагрузочное тестирование на базу `persons` с параметрами: 25 клиентов отправляло запросы в 8 потоков на протяжении 5 минут. 
После завершения теста будет выведен отчет:
```
pgbench (16.9 (Ubuntu 16.9-0ubuntu0.24.04.1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
query mode: simple
number of clients: 25
number of threads: 8
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 6279184
number of failed transactions: 0 (0.000%)
latency average = 1.195 ms
initial connection time = 10.725 ms
tps = 20928.976687 (without initial connection time)
```
Из которого видно, что текущий TPS (transaction per second) равен 20,928.

**Дисклеймер**: pgbench не дает абсолютного значения TPS при любых условиях и сильно зависит не только от производительности сервера, но 
и от выбранного профиля нагрузки. Так, если вы укажете слишком малое количество клиентов, то не задействуете всю производительность CPU 
и получите меньшее количество TSP на выходе. Поэтому, эмпирически выявлены следующие параметры: 

Клиенты (-с) = `max_connections - 5`, чтобы задействовать практически все доступные подключения
Потоки (-j) = `max_worker_processes * 2`, чтобы задействовать hyperthreading
Время теста (-T) = 300-600, чтобы получить более точные результаты

Если же менять параметры в большую или меньшую сторону, то значение TSP может упасть в два и более раз.

### Скриншоты во время теста
Команда htop:
![htop.png](img/htop.png)
Графики в PgAdmin4:
![pgadmin-dashboard.png](img/pgadmin-dashboard.png)

