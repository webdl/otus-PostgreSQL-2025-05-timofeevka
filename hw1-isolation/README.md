# Работа с уровнями изоляции транзакции в PostgreSQL

Подключился к PG двумя разными сессиями.

В первой сессии создать и наполнить таблицу для дальнейшей работы:
```sql
BEGIN;
CREATE TABLE persons
(
    id          serial,
    first_name  text,
    second_name text
);
INSERT INTO persons(first_name, second_name) VALUES ('ivan', 'ivanov');
INSERT INTO persons(first_name, second_name) VALUES ('petr', 'petrov');
COMMIT;
```

Посмотреть текущий уровень изоляции транзакции:
```sql
SHOW transaction_isolation;

--  transaction_isolation 
-- -----------------------
--  read committed
-- (1 row)
```

Начать новую транзакцию в обеих сессиях:
```sql
BEGIN;
```

В первой сессии добавить новую запись без закрытия транзакции:
```sql
INSERT INTO persons(first_name, second_name) VALUES ('sergey', 'sergeev');
```

Во второй сессии посмотреть все записи без закрытия транзакции:
```sql
SELECT * FROM persons;

-- id | first_name | second_name 
-- ----+------------+-------------
--   1 | ivan       | ivanov
--   2 | petr       | petrov
-- (2 rows)
```

**Вопрос:** видите ли вы новую запись и если да то почему?  
**Ответ:** Нет, потому что транзакция в первой сессии ещё не завершена.

Завершить транзакцию в первой сессии:
```sql
COMMIT;
```

Полный код из первой сессии:
```sql
BEGIN;
INSERT INTO persons(first_name, second_name) VALUES ('sergey', 'sergeev');
COMMIT;
```

Во второй сессии повторить запрос и увидеть новую запись:
```sql
SELECT * FROM persons;

-- id | first_name | second_name 
-- ----+------------+-------------
--   1 | ivan       | ivanov
--   2 | petr       | petrov
--   3 | sergey     | sergeev
-- (3 rows)
```

**Вопрос:** видите ли вы новую запись и если да то почему?  
**Ответ:** Да, потому что транзакция в первой сессии была завершена, а текущий уровень изоляции транзакций во второй сессии допускает 
неповторяющееся чтение

Завершить транзакцию во второй сессии:
```sql
COMMIT;
```
---
Во второй сессии начать новую транзакцию с другим уровнем изоляции:
```sql
BEGIN;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

В первой сессии нет необходимости в другом уровне изоляции, можно сразу запустить вставку новой записи:
```sql
BEGIN;
INSERT INTO persons(first_name, second_name) VALUES ('sveta', 'svetova');
```

Во второй сессии прочитать всю таблицу. Новых записей не появилось:
```sql
SELECT * FROM persons;

-- id | first_name | second_name 
-- ----+------------+-------------
--   1 | ivan       | ivanov
--   2 | petr       | petrov
--   3 | sergey     | sergeev
-- (3 rows)
```

**Вопрос:** видите ли вы новую запись и если да то почему?  
**Ответ:** Нет, потому что транзакция в первой сессии ещё не завершена.

Завершить транзакцию в первой сессии:
```sql
COMMIT;
```

Во второй сессии повторить запрос, но новая запись не появится:
```sql
SELECT * FROM persons;

-- id | first_name | second_name 
-- ----+------------+-------------
--   1 | ivan       | ivanov
--   2 | petr       | petrov
--   3 | sergey     | sergeev
-- (3 rows)
```

**Вопрос:** видите ли вы новую запись и если да то почему?  
**Ответ:** Нет, потому что уровень изоляции REPEATABLE READ гарантирует, что в рамках текущей транзакции нам доступны данные из снимка 
данных, созданного на момент начала транзакции.

Завершить транзакцию во второй сессии и повторите запрос:
```sql
COMMIT;
SELECT * FROM persons;

-- id | first_name | second_name 
-- ----+------------+-------------
--   1 | ivan       | ivanov
--   2 | petr       | petrov
--   3 | sergey     | sergeev
--   4 | sveta      | svetova
-- (4 rows)
```

**Вопрос:** видите ли вы новую запись и если да то почему?  
**Ответ:** Да, потому что вышли из REPEATABLE READ транзакции и получили самые актуальные данных в таблице.
