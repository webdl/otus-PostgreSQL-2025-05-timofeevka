# Резервное копирование и восстановление

Цели:

* применить логический бэкап. Восстановиться из бэкапа

# Решение домашнего задания

## Подготовка БД для РК

Создадим БД и таблицу, резервную копию которых будет производить.

```sql
CREATE DATABASE backups;

-- переключитесь на БД backups
CREATE TABLE IF NOT EXISTS users (
	id SERIAL PRIMARY KEY,
	name VARCHAR(255) NOT NULL,
	email VARCHAR(255) NOT NULL UNIQUE
);

INSERT INTO users (name, email)
SELECT 
	'User' || generate_series AS name,
	'User' || generate_series || '@example.com' AS email
FROM generate_series(1, 100);

SELECT * FROM users;
```

## Резервное копирование и восстановление с помощью COPY

На сервере БД создадим каталог под бекапы.

```shell
sudo su - postgres
mkdir ~/backups
cd ~/backups
pwd
```

Выполним РК

```sql
COPY users TO '/var/lib/postgresql/backups/users.csv' WITH DELIMITER '|' HEADER;
```

Выполняем восстановление в другую таблицу

```sql
CREATE TABLE users_copy (LIKE users INCLUDING ALL);

COPY users_copy FROM '/var/lib/postgresql/backups/users.csv' WITH DELIMITER '|' HEADER;

SELECT * FROM users_copy;
```
