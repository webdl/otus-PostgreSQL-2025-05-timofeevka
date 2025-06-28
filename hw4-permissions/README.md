# Работа с базами данных, пользователями и правами
### Цели:
1. создание новой базы данных, схемы и таблицы
2. создание роли для чтения данных из созданной схемы созданной базы данных
3. создание роли для чтения и записи из созданной схемы созданной базы данных

# Решение домашнего задания
С помощью DBeaver или PgAdmin4 подключиться к ранее созданному кластеру `pg-master` под учетной записью `postgres`.

Создать новую БД:
```sql
CREATE DATABASE testdb;
```

Переключиться на созданную БД и создать схему:
```sql
CREATE SCHEMA IF NOT EXISTS testnm;
SET search_path = testnm;
```

Создать таблицу с данными:
```sql
START TRANSACTION;
CREATE TABLE IF NOT EXISTS t1 (c1 int);
INSERT INTO t1(c1) VALUES (1);
COMMIT;
```

Создать новую роль `readonly` и выдать ей права:
```sql
START TRANSACTION;
CREATE ROLE readonly;
GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;

-- Проверка
SELECT *
FROM information_schema.role_table_grants
WHERE grantee = 'readonly';

COMMIT;
```

Создать пользователя `testread`:
```sql
START TRANSACTION;
CREATE USER testread WITH PASSWORD 'test123' IN ROLE readonly;
COMMIT;
```

Подключиться по SSH к серверу и проверить доступ пользователя к данным в таблице `t1`:
```shell
> psql -U testread -d testdb -h localhost -c "select * from testnm.t1;"
Password for user testread:
 c1
----
  1
(1 row)
```
Обратите внимание: ключ `-h localhost` требуется, чтобы подключение к серверу происходило по сети, как указано в настройках 
[pg_hba](../deploy/vm/group_vars/db.yml). В противном случае будет ошибка:
```
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
```

**Важно!** Пункты 22-36 домашнего задания пропущены, так как изначально была выстроена правильная работа со схемами и описанные в 
пунктах проблемы не возникли. 

Однако для закрепления материала стоит отметить, что указанной выше команды для предоставления доступа на чтение всех таблиц в схеме 
`testnm` **недостаточно**, так как он распространяется только на текущие созданные таблицы.

Чтобы выдать доступ ко всем будущим таблицам, нужно задать поведение по-умолчанию:
```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES to readonly;
```
---
Провести эксперимент и ответить на вопрос: может ли пользователь `testread` создавать новые таблицы?

Подключиться по SSH к серверу и выполнить команду:
```shell
> psql -U testread -d testdb -h localhost -c "CREATE TABLE t2(c1 int); INSERT INTO t2(c1) VALUES(2);"
Password for user testread:
ERROR:  permission denied for schema public
LINE 1: CREATE TABLE t2(c1 int); INSERT INTO t2(c1) VALUES(2);
                     ^
```
Как видим из вывода консоли, создание таблиц запрещено даже в схеме `public`. Однако так было не всегда: начиная с PostgreSQL 15+, право
`CREATE TABLE` по умолчанию отозвано у схемы `PUBLIC` (оставлен только `USAGE`). Поскольку у нас используется 16-я версия, система 
работает ожидаемо.
