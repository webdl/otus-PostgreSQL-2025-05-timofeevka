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
progress: 6.0 s, 6109.2 tps, lat 1.301 ms stddev 0.765, 0 failed
progress: 12.0 s, 5649.7 tps, lat 1.411 ms stddev 0.829, 0 failed
progress: 18.0 s, 5553.2 tps, lat 1.436 ms stddev 0.851, 0 failed
progress: 24.0 s, 5464.1 tps, lat 1.459 ms stddev 0.839, 0 failed
progress: 30.0 s, 5341.8 tps, lat 1.493 ms stddev 0.882, 0 failed
progress: 36.0 s, 5431.1 tps, lat 1.468 ms stddev 0.835, 0 failed
progress: 42.0 s, 5313.7 tps, lat 1.501 ms stddev 0.871, 0 failed
progress: 48.0 s, 5904.4 tps, lat 1.350 ms stddev 0.801, 0 failed
progress: 54.0 s, 5970.7 tps, lat 1.335 ms stddev 0.785, 0 failed
progress: 60.0 s, 6043.0 tps, lat 1.319 ms stddev 0.778, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 340694
number of failed transactions: 0 (0.000%)
latency average = 1.404 ms
latency stddev = 0.826 ms
initial connection time = 18.731 ms
tps = 5679.624179 (without initial connection time)
```

Применить настройки тюнинга выше и повторить тест:
```shell
> pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (17.5 (Debian 17.5-1.pgdg120+1))
starting vacuum...end.
progress: 6.0 s, 5570.4 tps, lat 1.428 ms stddev 0.893, 0 failed
progress: 12.0 s, 5360.4 tps, lat 1.487 ms stddev 0.918, 0 failed
progress: 18.0 s, 5763.6 tps, lat 1.384 ms stddev 0.817, 0 failed
progress: 24.0 s, 5838.7 tps, lat 1.365 ms stddev 0.802, 0 failed
progress: 30.0 s, 5952.5 tps, lat 1.339 ms stddev 0.809, 0 failed
progress: 36.0 s, 5942.2 tps, lat 1.342 ms stddev 0.809, 0 failed
progress: 42.0 s, 5648.2 tps, lat 1.412 ms stddev 0.850, 0 failed
progress: 48.0 s, 5678.5 tps, lat 1.404 ms stddev 0.807, 0 failed
progress: 54.0 s, 5903.0 tps, lat 1.350 ms stddev 0.810, 0 failed
progress: 60.0 s, 5898.0 tps, lat 1.352 ms stddev 0.793, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 345343
number of failed transactions: 0 (0.000%)
latency average = 1.385 ms
latency stddev = 0.832 ms
initial connection time = 14.112 ms
tps = 5756.490531 (without initial connection time)
```
### Выводы
Выводы к предыдущему эксперименту оказались неверные, так как при изменении среды виртуализации производительность БД после тюнинга так 
же не выросла... От чего делаю вывод, что изменение указанных параметров не приводит к улучшению производительности для указанного 
вызова pgbench (прошу проверяющих дать комментарии, если я где-то ошибся).

## Создание таблицы с текстовыми данными
**Дисклеймер:** Дальнейшие работы проводятся на ранее созданной ВМ.

Для следующего теста потребуется создать таблицу с одним текстовым полем и заполнить ее 1 млн. записей. Для этого можно воспользоваться 
SQL запросами:
```sql
CREATE DATABASE autovacuum;
-- Use autovacuum

CREATE TABLE bigdata (
    data TEXT NOT NULL
);

INSERT INTO bigdata (data)
SELECT md5(random()::text)
FROM generate_series(1, 1000000);
```

## Проверка работы autovacuum

Найти путь до файла с нашей таблицей:
```sql
SELECT pg_relation_filepath('bigdata');
pg_relation_filepath 
----------------------
 base/24603/24604
(1 row)
```

Зайти на ВМ по SSH и посмотреть размер файла:
```shell
> du -sh /mnt/pg_data/main/base/24603/24604
66M	/mnt/pg_data/main/base/24603/24604
```

Или через SQL запрос:
```sql
SELECT pg_size_pretty(pg_relation_size('bigdata'));
pg_size_pretty 
----------------
 65 MB
(1 row)
```

Посмотрит на текущее количество "мертвых" записей в таблице с помощью запроса ниже. Сейчас он покажет записей, но пригодится позднее, 
где я по тексту буду ссылаться на него.
```sql
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum 
FROM pg_stat_user_tables 
WHERE relname = 'bigdata';
```

Обновляем 5 раз все записи в таблице, добавляя туда по одному символу:
```sql
UPDATE bigdata
SET data = data || 'a';

UPDATE bigdata
SET data = data || 'b';

UPDATE bigdata
SET data = data || 'c';

UPDATE bigdata
SET data = data || 'd';

UPDATE bigdata
SET data = data || 'e';
```

С помощью запроса выше посмотреть на кол-во "мертвых" записей и время последнего запуска autovacuum:
```
relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 bigdata |    1000000 |    5000000 |    499 | 2025-07-05 07:57:23.449195+00
(1 row)
```

И текущий размер таблицы:
```sql
SELECT pg_size_pretty(pg_relation_size('bigdata'));
 pg_size_pretty 
----------------
 391 MB
(1 row)
```
Подождать некоторое время, пока не пройдет autovacuum и посторить запрос:
```
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 bigdata |    1000000 |          0 |      0 | 2025-07-05 08:10:25.701237+00
(1 row)
```
Autovaccum прошел. А что с размером таблицы?
```sql
SELECT pg_size_pretty(pg_relation_size('bigdata'));
 pg_size_pretty 
----------------
 391 MB
(1 row)
```
Он не изменился.

## Отключение autovacuum

Используя запросы выше повторно обновить 5 раз таблицу с данными:
```sql
UPDATE bigdata
SET data = data || 'a';

UPDATE bigdata
SET data = data || 'b';

UPDATE bigdata
SET data = data || 'c';

UPDATE bigdata
SET data = data || 'd';

UPDATE bigdata
SET data = data || 'e';
```
Затем отключить autovacuum:
```sql
ALTER TABLE bigdata SET (autovacuum_enabled = false);
```
И выполнить еще 10 обновлений таблицы (см. выше).

Посмотреть на "мертвые" записи:
```
relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 bigdata |    1000000 |   14999937 |   1499 | 2025-07-05 08:10:25.701237+00
(1 row)
```
И сколько занимает таблица:
```sql
SELECT pg_size_pretty(pg_relation_size('bigdata'));
 pg_size_pretty 
----------------
 1182 MB
(1 row)
```

## Выводы
Таблица непрерывно растет в размерах потому, что процесс обновления записей в PostgreSQL создает физическую копию строк с обновленными 
данными, но не удаляет исходные строки. Исходные же строки помечаются специальным флагом и игнорируются при последующих операциях 
SELECT / UPDATE / DELETE. Это специфика работы MVCC в PostgreSQL, которая позволяет очень быстро делать ROLLBACK для обновленных записей,
так как для отката изменений не требуется делать INSERT, а только выставить нужных флаг.

При этом схожим образом работает команда DELETE - записи не удаляются, а помечаются специальным флагом. Таким образом команда Update - это 
DELETE и INSERT в одном флаконе.

Однако это еще не конец, ведь старые записи всё еще находятся в БД. На этот случай и нужен vacuum - он удаляет данных из старых записей, 
отмеченных флагом. Но... Не освобождает место на диске. Читайте в следующем разделе почему так происходит. 

## Очищаем место на диске
PostgreSQL хранит свои данным в файлах на диске (ранее мы находили путь до файла таблицы `bigdata` и смотрели его размер). Для 
оптимизации работы базы данных после того, как выполнился autovacuum, файл не сжимается. Фактически происходит фрагментация данных - на 
месте удаленных записей остается пустота, и есть как минимум два способа с этим бороться.

### 1. Ничего не делать
Да, вы все правильно поняли. Дело в том, что разработчики PostgreSQL предусмотрели ситуации с фрагментацией, поэтому последующие команды 
INSERT будут добавлять данные в те самые пустые блоки, оставшиеся после работы vacuum. Это работает не каждый раз, имеет какой-то 
внутренний алгоритм, но это легко проверить.

Для начала надо включить обратно autovacuum и убедиться, что в таблице 1 млн записей и она занимает 1182 MB:
```sql
ALTER TABLE bigdata SET (autovacuum_enabled = true);
ANALYZE bigdata; -- собрать статистику по таблице
VACUUM bigdata; -- запустить ручной vacuum

SELECT count(*) FROM bigdata;
 count  
---------
 1000000
(1 row)

SELECT pg_size_pretty(pg_relation_size('bigdata'));
 pg_size_pretty 
----------------
 1182 MB
(1 row)

```
Теперь достаточно вставить новые данные в таблицу и посмотреть, как изменится размер:
```sql
INSERT INTO bigdata (data)
SELECT md5(random()::text)
FROM generate_series(1, 1000000);

SELECT pg_size_pretty(pg_relation_size('bigdata'));
 pg_size_pretty 
----------------
 1182 MB
(1 row)
```
Как можно заметить - размер не изменился, что подтверждает сказанное выше.

### 2. Принудительная дефрагментация (vacuum full)
Команда vacuum full делает следующее: 
1. создает новый файл таблицы на жестком диске
2. копирует "живые" данные из старого файла
3. изменяет ссылку на файл таблице в PostgreSQL
4. удаляет старый файл

Имеет смысл, если:
1. требуется срочное освобождение дискового пространства
2. появились проблемы с производительностью, связанные с сильной фрагментацией данных
3. проводились массовые удаления или обновления данных

**Важно отметить**, что во время работы vacuum full происходит полная блокировка доступ к таблице, а сама операция может занимать 
длительное время, если данных много. Поэтому выполнять ее нужно только в подходящее для этого время.

Проведем эксперимент: на нашей таблице запустить vacuum full и проверить ее новый размер. Для этого выполнить:
```sql
VACUUM FULL bigdata;
Query returned successfully in 425 msec.

SELECT pg_size_pretty(pg_relation_size('bigdata'));
 pg_size_pretty 
----------------
 146 MB
(1 row)
```
И убедиться, что путь до файла изменился:
```sql
SELECT pg_relation_filepath('bigdata');
pg_relation_filepath 
----------------------
 base/24603/24663 -- ранее был base/24603/24604
(1 row)
```
