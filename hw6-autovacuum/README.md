# Настройка autovacuum с учетом особенностей производительности
Цели:
* запустить нагрузочный тест pgbench
* настроить параметры autovacuum
* проверить работу autovacuum

# Решение домашнего задания
Исходные данные:
* Виртуальная машина: Ubuntu 24.04.2 LTS, 4 Gb RAM, 2 CPU, 10 Gb SSD
* PostgreSQL: 16.9

ВМ собрана на основе файлов настроек: [mini_db.yml](../deploy/vm/group_vars/mini_db.yml), [inventory.ini](../deploy/vm/inventory.ini)

## Нагрузочное тестирование сервера (неудачное)
Выполнить вход на ВМ по SSH. Создать БД для тестов и провести тестирование:
```shell
> sudo su - postgres

> pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.03 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.07 s (drop tables 0.00 s, create tables 0.00 s, client-side generate 0.04 s, vacuum 0.01 s, primary keys 0.02 s).

> pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (16.9 (Ubuntu 16.9-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 6.0 s, 4315.5 tps, lat 1.850 ms stddev 1.324, 0 failed
progress: 12.0 s, 4336.5 tps, lat 1.843 ms stddev 1.314, 0 failed
progress: 18.0 s, 4323.5 tps, lat 1.849 ms stddev 1.291, 0 failed
progress: 24.0 s, 4273.3 tps, lat 1.871 ms stddev 1.332, 0 failed
progress: 30.0 s, 4162.5 tps, lat 1.920 ms stddev 1.409, 0 failed
progress: 36.0 s, 4338.0 tps, lat 1.843 ms stddev 1.313, 0 failed
progress: 42.0 s, 4450.5 tps, lat 1.796 ms stddev 1.266, 0 failed
progress: 48.0 s, 4313.3 tps, lat 1.853 ms stddev 1.324, 0 failed
progress: 54.0 s, 4331.8 tps, lat 1.845 ms stddev 1.316, 0 failed
progress: 60.0 s, 4335.0 tps, lat 1.844 ms stddev 1.306, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 259088
number of failed transactions: 0 (0.000%)
latency average = 1.851 ms
latency stddev = 1.320 ms
initial connection time = 8.228 ms
tps = 4318.365445 (without initial connection time)
```
### Тюнинг параметров
Применить к серверу следующие параметры:
```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```
Для этого они были помещены в [mini_db.yml](../deploy/vm/group_vars/mini_db.yml), а для применения надо перейти в локальный каталог 
deploy/vm и выполнить команду:
```shell
> ansible-playbook playbooks/install_db.yml -l pg-slave01
```
Выполнить вход на ВМ по SSH и протестировать повторно:
```shell
> sudo su - postgres

> pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (16.9 (Ubuntu 16.9-0ubuntu0.24.04.1))
starting vacuum...end.
progress: 6.0 s, 4291.7 tps, lat 1.860 ms stddev 1.331, 0 failed
progress: 12.0 s, 4337.3 tps, lat 1.843 ms stddev 1.312, 0 failed
progress: 18.0 s, 4358.5 tps, lat 1.834 ms stddev 1.285, 0 failed
progress: 24.0 s, 4376.2 tps, lat 1.827 ms stddev 1.291, 0 failed
progress: 30.0 s, 4338.3 tps, lat 1.843 ms stddev 1.313, 0 failed
progress: 36.0 s, 4399.5 tps, lat 1.817 ms stddev 1.322, 0 failed
progress: 42.0 s, 4360.6 tps, lat 1.833 ms stddev 1.293, 0 failed
progress: 48.0 s, 4298.8 tps, lat 1.860 ms stddev 1.305, 0 failed
progress: 54.0 s, 4351.3 tps, lat 1.837 ms stddev 1.303, 0 failed
progress: 60.0 s, 4178.8 tps, lat 1.913 ms stddev 1.353, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 259755
number of failed transactions: 0 (0.000%)
latency average = 1.846 ms
latency stddev = 1.311 ms
initial connection time = 9.395 ms
tps = 4329.449660 (without initial connection time)
```
### Выводы
Сравнивая результаты с предыдущим видно, что ничего не изменилось. Возможно это связано со спецификой виртуализации UTM на MacOS, так 
как в прошлых экспериментах уже были замечены незначительные улучшения производительности при тюнинге настроек. Поэтому требуется 
повторное тестирование в другой среде.

## Нагрузочное тестирование сервера (неудачное, снова...)
С помощью [docker-compose.yml](../deploy/docker/mini_db/docker-compose.yml) разворачивает PostgreSQL в Docker. Для этого надо перейти в 
каталог deploy/docker/mini_db и выполнить команду:
```shell
docker-compose up -d
```
После запуска контейнера провести тот же тест:
```shell
> su postgres

> pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...                                                                              
creating primary keys...
done in 0.11 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.06 s, vacuum 0.02 s, primary keys 0.04 s).

> pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (17.5 (Debian 17.5-1.pgdg120+1))
starting vacuum...end.
progress: 6.0 s, 278.3 tps, lat 28.450 ms stddev 39.746, 0 failed
progress: 12.0 s, 279.3 tps, lat 28.556 ms stddev 39.711, 0 failed
progress: 18.0 s, 290.3 tps, lat 27.753 ms stddev 39.122, 0 failed
progress: 24.0 s, 283.8 tps, lat 28.128 ms stddev 39.393, 0 failed
progress: 30.0 s, 285.7 tps, lat 27.941 ms stddev 39.517, 0 failed
progress: 36.0 s, 281.7 tps, lat 28.442 ms stddev 39.567, 0 failed
progress: 42.0 s, 286.0 tps, lat 28.009 ms stddev 39.555, 0 failed
progress: 48.0 s, 297.5 tps, lat 26.941 ms stddev 38.808, 0 failed
progress: 54.0 s, 296.5 tps, lat 26.807 ms stddev 38.801, 0 failed
progress: 60.0 s, 268.8 tps, lat 29.752 ms stddev 40.806, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 17096
number of failed transactions: 0 (0.000%)
latency average = 28.067 ms
latency stddev = 39.519 ms
initial connection time = 14.851 ms
tps = 284.539777 (without initial connection time)
```
Применить настройки тюнинга выше и повторить тест:
```shell
> pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (17.5 (Debian 17.5-1.pgdg120+1))
starting vacuum...end.
progress: 6.0 s, 270.3 tps, lat 29.502 ms stddev 40.254, 0 failed
progress: 12.0 s, 263.7 tps, lat 30.269 ms stddev 40.747, 0 failed
progress: 18.0 s, 249.0 tps, lat 32.176 ms stddev 41.895, 0 failed
progress: 24.0 s, 264.0 tps, lat 30.300 ms stddev 40.825, 0 failed
progress: 30.0 s, 286.3 tps, lat 27.825 ms stddev 39.737, 0 failed
progress: 36.0 s, 330.2 tps, lat 24.221 ms stddev 37.917, 0 failed
progress: 42.0 s, 274.3 tps, lat 29.261 ms stddev 41.113, 0 failed
progress: 48.0 s, 252.2 tps, lat 31.655 ms stddev 40.793, 0 failed
progress: 54.0 s, 277.3 tps, lat 28.843 ms stddev 40.179, 0 failed
progress: 60.0 s, 301.2 tps, lat 26.451 ms stddev 38.886, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 16619
number of failed transactions: 0 (0.000%)
latency average = 28.890 ms
latency stddev = 40.247 ms
initial connection time = 13.571 ms
tps = 276.588890 (without initial connection time)
```
### Выводы
Выводы к предыдущему эксперименту оказались неверные, так как при изменении среды виртуализации производительность БД после тюнинга так 
же не выросла... От чего делаю вывод, что изменение указанных параметров не приводит к улучшению производительности для указанного 
вызова pgbench (прошу проверяющих дать комментарии, если я где-то ошибся).
