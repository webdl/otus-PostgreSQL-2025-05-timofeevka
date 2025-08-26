# Логическая и физическая репликации

Цель: реализовать различные виды репликаций на 4 ВМ.

# Подготовка к решению домашнего задания

## Перенастройка старых ВМ

Ранее мы использовали 2 ВМ в наших экспериментах, а теперь к ним необходимо добавить еще 2 ВМ для организации кластера PG. Перед этим
выровним настройки по текущим ВМ:

Первая ВМ была "жирная", с большим количеством памяти и CPU, так как ранее мы активно проводили замеры производительности. Через систему
управления ВМ приводим ее настройки к параметрам: CPU = 2, RAM = 4. Этих значений хватит для экспериментальной настройки репликации.

Вводим новые параметры ВМ в сервис https://pgtune.leopard.in.ua/ и применяем полученные настройки в файле
[db.yml](../deploy/vm/group_vars/db.yml). Изменения следующие:

```
max_connections = 20
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
work_mem = 37449kB
max_worker_processes = 2
max_parallel_workers = 2
checkpoint_timeout = 60
deadlock_timeout = 1000ms
```

В файле [inventory.ini](../deploy/vm/inventory.ini) переносим хост `pg-slave01` в группу `db`, чтобы управлять им одной конфигурацией.
Файл [mini_db.yml](../deploy/vm/group_vars/mini_db.yml) удаляем, так как он теряет свою актуальность.

Запускаем две ВМ и выполняем команду перенастройки систем:

```shell
cd deploy/vm
ansible-playbook playbooks/main.yml
```

Если всё прошло хорошо — можно приступить к следующему разделу.

## Создание новых ВМ

Путем клонирования ВМ `pg-slave01` создаем `pg-slave02` и `pg-slave03`. Перенастраиваем их имена, IP адреса и т.д. Выполняем команду
ниже и убеждаемся, что сервера в рабочем состоянии с единой конфигурацией:

```shell
ansible-playbook playbooks/main.yml
```

```
PLAY RECAP **********************************************************************************************************
pg-master                  : ok=36   changed=1    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0   
pg-slave01                 : ok=36   changed=1    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0   
pg-slave02                 : ok=36   changed=1    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0   
pg-slave03                 : ok=36   changed=1    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0
```

# Решение домашнего задания

## Логическая репликация

### Настройка серверов PostgreSQL

Для работы логической репликации требуется поменять параметр `wal_level` в файле `postgresql.conf`, который принимает следующие значения:

* minimal — минимальный уровень журналирования, записывается только необходимая информация для восстановления после сбоев. Используется
  без репликации и архивирования WAL, позволяет ускорить некоторые операции, но не поддерживает репликацию.
* replica (по умолчанию) — записывается информация, необходимая для поддержки репликации и архивирования WAL, включая возможность чтения
  только с ведомой базы.
* logical — самый высокий уровень, записывается дополнительная информация, необходимая для логического декодирования изменений и
  логической репликации.

Обратите внимание:
Каждый следующий уровень включает информацию всех предыдущих. Настройка этого параметра влияет на производительность, объем генерируемых
WAL и функциональность репликации и резервного копирования.

Для настройки логической репликации устанавливаем его в значение `logical` через [db.yml](../deploy/vm/group_vars/db.yml) и применяем ко
всем ВМ.

```shell
ansible-playbook playbooks/main.yml
```

```
changed: [pg-slave02] => (item={'option': 'wal_level', 'value': 'logical'})
--- before: /etc/postgresql/16/main/postgresql.conf (content)
+++ after: /etc/postgresql/16/main/postgresql.conf (content)
@@ -206,11 +206,11 @@
 # WRITE-AHEAD LOG
 #------------------------------------------------------------------------------
 
 # - Settings -
 
-#wal_level = replica                   # minimal, replica, or logical
+wal_level = 'logical'
...
RUNNING HANDLER [geerlingguy.postgresql : restart postgresql] ******************************************************
[WARNING]: Ignoring "sleep" as it is not used in "systemd"
changed: [pg-slave02]
changed: [pg-master]
changed: [pg-slave01]
changed: [pg-slave03]
...
PLAY RECAP **********************************************************************************************************
pg-master                  : ok=37   changed=3    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0   
pg-slave01                 : ok=37   changed=3    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0   
pg-slave02                 : ok=37   changed=3    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0   
pg-slave03                 : ok=37   changed=3    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0
```

### Настройка логической репликации

На `pg-master` и `pg-slave01` создаем БД, таблицы в ней и УЗ для репликации:

```sql
CREATE DATABASE replica;

CREATE TABLE table1 (
	id SERIAL PRIMARY KEY,
	name TEXT
);

CREATE TABLE table2 (
	id SERIAL PRIMARY KEY,
	name TEXT
);

CREATE ROLE replica_user WITH
	LOGIN
	REPLICATION
	PASSWORD 'replica_user';

GRANT SELECT ON TABLE public.table1 TO replica_user;
GRANT SELECT ON TABLE public.table2 TO replica_user;
```

Настраиваем публикацию `table1` на сервере `pg-master`:

```sql
CREATE PUBLICATION table1
    FOR TABLE public.table1
    WITH (publish = 'insert, update, delete, truncate', publish_via_partition_root = false);
```

Настраиваем на сервере `pg-slave01` подписку на `table1` с `pg-master`:

```sql
CREATE SUBSCRIPTION pg_master_table1
    CONNECTION 'host=192.168.0.240 port=5432 user=replica_user password=replica_user dbname=replica connect_timeout=10 sslmode=prefer'
    PUBLICATION table1
    WITH (connect = true, enabled = true, copy_data = true, create_slot = true, synchronous_commit = 'off', binary = false, 
    streaming = 'False', two_phase = false, disable_on_error = false, run_as_owner = false, password_required = true, origin = 'any');
```

Настраиваем публикацию `table2` на `pg-slave01`:

```sql
CREATE PUBLICATION table2
    FOR TABLE public.table2
    WITH (publish = 'insert, update, delete, truncate', publish_via_partition_root = false);
```

Настраиваем на сервере `pg-master` подписку на `table2` с `pg-slave01`:

```sql
CREATE SUBSCRIPTION pg_slave01_table2
    CONNECTION 'host=192.168.0.241 port=5432 user=replica_user password=replica_user dbname=replica connect_timeout=10 sslmode=prefer'
    PUBLICATION table2
    WITH (connect = true, enabled = true, copy_data = true, create_slot = true, synchronous_commit = 'off', binary = false, 
    streaming = 'False', two_phase = false, disable_on_error = false, run_as_owner = false, password_required = true, origin = 'any');
```

### Проверка репликации pg-master -> pg-slave01

Смотрим статус подписки на `pg-slave01`:

```sql
SELECT * FROM pg_stat_subscription;
```

```
 subid |     subname      | pid  | leader_pid | relid | received_lsn |      last_msg_send_time      |     last_msg_receipt_time     | latest_end_lsn |       latest_end_time        
-------+------------------+------+------------+-------+--------------+------------------------------+-------------------------------+----------------+------------------------------
 41015 | pg_master_table1 | 9785 |            |       | 3F/91A8B9D0  | 2025-08-25 17:18:24.67156+00 | 2025-08-25 17:18:24.660765+00 | 3F/91A8B9D0    | 2025-08-25 17:18:24.67156+00
```

На `pg-master` выполняем вставку:

```sql
INSERT INTO table1 (name) VALUES ('Some name');
INSERT INTO table1 (name) VALUES ('Some name2');
INSERT INTO table1 (name) VALUES ('Some name3');

SELECT * FROM table1;
```

```
 id |    name    
----+------------
  1 | Some name
  2 | Some name2
  3 | Some name3
```

На `pg-slave01` выполняем выборку:

```sql
SELECT * FROM table1;
```

```
 id |    name    
----+------------
  1 | Some name
  2 | Some name2
  3 | Some name3
```

Логическая репликация работает!

### Проверка репликации pg-slave01 -> pg-master

Смотрим статус подписки на `pg-master`:

```sql
SELECT * FROM pg_stat_subscription;
```

```
 subid |      subname      |  pid  | leader_pid | relid | received_lsn |      last_msg_send_time       |     last_msg_receipt_time     | latest_end_lsn |        latest_end_time        
-------+-------------------+-------+------------+-------+--------------+-------------------------------+-------------------------------+----------------+-------------------------------
 84865 | pg_slave01_table2 | 10081 |            |       | 2/CEA58970   | 2025-08-25 17:37:37.449742+00 | 2025-08-25 17:37:37.470897+00 | 2/CEA58970     | 2025-08-25 17:37:37.449742+00
```

На `pg-slave01` выполняем вставку:

```sql
INSERT INTO table2 (name) VALUES ('Some name 1');
INSERT INTO table2 (name) VALUES ('Some name 2');
INSERT INTO table2 (name) VALUES ('Some name 3');

SELECT * FROM table2;
```

```
 id |    name    
----+-------------
  1 | Some name 1
  2 | Some name 2
  3 | Some name 3
```

На `pg-master` выполняем выборку:

```sql
SELECT * FROM table2;
```

```
 id |    name    
----+-------------
  1 | Some name 1
  2 | Some name 2
  3 | Some name 3
```

Логическая репликация работает!

### Настройка третьего сервера

Третью ВМ будем использовать как реплику для чтения и бэкапов и настроим на ней подписку на `pg-master` и `pg-slave01`:

```sql
CREATE DATABASE replica;

CREATE TABLE table1 (
	id SERIAL PRIMARY KEY,
	name TEXT
);

CREATE TABLE table2 (
	id SERIAL PRIMARY KEY,
	name TEXT
);

CREATE ROLE replica_user WITH
	LOGIN
	REPLICATION
	PASSWORD 'replica_user';

GRANT SELECT ON TABLE public.table1 TO replica_user;
GRANT SELECT ON TABLE public.table2 TO replica_user;

CREATE SUBSCRIPTION pg_slave02_table1
    CONNECTION 'host=192.168.0.240 port=5432 user=replica_user password=replica_user dbname=replica connect_timeout=10 sslmode=prefer'
    PUBLICATION table1
    WITH (connect = true, enabled = true, copy_data = true, create_slot = true, synchronous_commit = 'off', binary = false, 
    streaming = 'False', two_phase = false, disable_on_error = false, run_as_owner = false, password_required = true, origin = 'any');

CREATE SUBSCRIPTION pg_slave01_table2
    CONNECTION 'host=192.168.0.241 port=5432 user=replica_user password=replica_user dbname=replica connect_timeout=10 sslmode=prefer'
    PUBLICATION table2
    WITH (connect = true, enabled = true, copy_data = true, create_slot = true, synchronous_commit = 'off', binary = false, 
    streaming = 'False', two_phase = false, disable_on_error = false, run_as_owner = false, password_required = true, origin = 'any');
```

# Задание со *

Реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами
столкнулись.

## Настройка мастер сервера

Мастер сервер для репликации у нас `pg-slave02`, пусть вас не путает название, так получилось. :)

Для репликации будет использоваться УЗ `replica_user`, которую мы создавали ранее. Чтобы она заработала, потребуется прописать ей доступ
в файле `pg_hba.conf` на сервере `pg-slave02`, откуда будет происходить репликация. Делаем это через переменные хостов
[pg-slave02](../deploy/vm/host_vars/pg-slave02.yml).

Настройка репликации на принимающем сервере `pg-slave03` делается в несколько действий через SSH.

Подключаемся к серверу под УЗ `postgres`:

```shell
sudo su - postgres
```

Останавливаем сервер:

```shell
pg_ctlcluster 16 main stop
```

Удаляем старый каталог с данными:

```shell
cd /mnt/pg_data/
rm -rf main
```

Клонируем с мастер-сервера каталог с данными:

```shell
pg_basebackup -h 192.168.0.242 -D /mnt/pg_data/main/ -U replica_user -P --wal-method=stream -R
```

Финальным штрихом прописываем переменную `primary_connifo` в файле `postgresql.conf` через
[pg-slave03](../deploy/vm/host_vars/pg-slave03.yml):

```
primary_conninfo = 'host=192.168.0.242 post=5432 user=replica_user password=replica_user'
```

Запускаем сервер:

```shell
pg_ctlcluster 16 main start
```

## Проверка репликации

Если всё настроено корректно, то проверяем следующим образом:

1. На сервере `pg-slave01` добавляем новую запись в `table2`
2. Она через логическую репликацию попадает на `pg-slave02`
3. А зачем через физическую репликацию оказывается на `pg-slave03`

## Трудности, с которыми столкнулся

### В pg_hba.conf обязательно должен быть доступ

Примечательно то, что для логической репликации прописывать эти доступы не требуется — она работает за счет доступа к БД. А вот
физическая репликация явно ругалась на отсутствие доступа.

### Важно использовать ключ -R для команды pg_basebackup

Ключ `-R` при копировании создает в каталоге данных файл `standby.signal`, которые сообщает PostgreSQL, что данный сервер находится в
режиме slave.

Я очень долго пытался понять в чем проблема, когда не использовал данных ключ. При этом кластер на `pg-slave03` поднимался успешно, но в
логах было много ошибок связанных с подпиской на публикации для логической репликации, так как имя для подписки использовалось такое же,
что на `pg-slave02`.
