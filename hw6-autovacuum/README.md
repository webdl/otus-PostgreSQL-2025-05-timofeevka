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

## Нагрузочное тестирование сервера
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
