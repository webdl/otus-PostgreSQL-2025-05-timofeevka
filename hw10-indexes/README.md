# Работа с индексами

Цели:

* знать и уметь применять основные виды индексов PostgreSQL
* строить и анализировать план выполнения запроса
* уметь оптимизировать запросы для с использованием индексов

# Подготовка к решению домашнего задания

## Функция генерации текста

Для использования GIN индекса используем простую функцию генерации lorem ipsum текста.

Установим требуемое расширение:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

Создадим функцию:

```sql

CREATE OR REPLACE FUNCTION lorem_ipsum(length bigint) RETURNS text AS $$
DECLARE
    result text;
BEGIN
    SELECT string_agg(word, ' ' ORDER BY gen_random_uuid()) INTO result
    FROM (
        SELECT UNNEST(ARRAY[
            'lorem', 'ipsum', 'dolor', 'sit', 'amet,', 'consectetur', 'adipiscing', 'elit,',
            'sed', 'do', 'eiusmod', 'tempor', 'incididunt', 'ut', 'labore', 'et', 'dolore',
            'magna', 'aliqua.', 'ut', 'enim', 'ad', 'minim', 'veniam,', 'quis', 'nostrud',
            'exercitation', 'ullamco', 'laboris', 'nisi', 'aliquip', 'ex', 'ea', 'commodo',
            'consequat.', 'duis', 'aute', 'irure', 'in', 'reprehenderit', 'voluptate', 'velit',
            'esse', 'cillum', 'eu', 'fugiat', 'nulla', 'pariatur.', 'excepteur', 'sint',
            'occaecat', 'cupidatat', 'non', 'proident,', 'sunt', 'culpa', 'qui', 'officia',
            'deserunt', 'mollit', 'anim', 'id', 'est', 'laborum'
        ]) AS word
    ) words;

    RETURN initcap(substring(result FROM 1 FOR length::int));
END;
$$ LANGUAGE plpgsql;
```

## Таблица с УЗ пользователей

Создадим основную таблицу, на которую будем накладывать индексы:

```sql
DROP TABLE IF EXISTS users;

CREATE TABLE IF NOT EXISTS users (
    id 			serial UNIQUE PRIMARY KEY,
    username    varchar(50) UNIQUE,
    email       varchar(100),
    age         int,
    rating      numeric(5,2),
    is_active   boolean,
    meta        jsonb,
    about       text,
    created_at 	timestamp
);
```

Наполним ее 3,000,000 сгенерированными данными (генерация занимает ~2,5 минуты):

```sql
INSERT INTO users (username, email, age, rating, is_active, meta, about, created_at)
SELECT 
    'user_' || gs,
    'user_' || gs || '@mail.ru',
	-- age: случайное число от 18 до 70
    18 + (random() * 52)::int, 			
	-- rating: случайное число от 0 до 100 с 2 знаками после запятой
    round((random() * 100)::numeric, 2),
	-- is_active: случайное true/false
    (random() > 0.2),
	-- meta: простой JSON объект с числом и строкой
    jsonb_build_object('score', round((random() * 10)::numeric, 2), 'status', CASE WHEN random() > 0.5 THEN 'ok' ELSE 'fail' END),
	-- about: случайный lorem ipsum текст
	lorem_ipsum(255),
	-- created_at: случайные даты с 2024 по середину 2025
    timestamp '2024-01-01 00:00:00' + (random() * 550)::int * interval '1 day'
FROM generate_series(1, 3000000) gs;
```

Пример полученных записей:

```
 id | username |      email      | age | rating | is_active |               meta                |                                                                                                                              about                                                                                                                              |     created_at      
----+----------+-----------------+-----+--------+-----------+-----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------
  1 | user_1   | user_1@mail.ru  |  55 |  56.51 | t         | {"score": 2.38, "status": "ok"}   | Excepteur Cillum Ullamco Ipsum Eiusmod Nisi Et Fugiat Exercitation Laboris Anim Dolore Sunt Commodo Aute Consequat. Officia Incididunt Pariatur. Mollit Qui Magna Id Reprehenderit Quis Sed Sint Eu Irure Culpa Laborum Occaecat Proident, Duis Adipiscing Labo | 2025-01-28 00:00:00
  2 | user_2   | user_2@mail.ru  |  19 |  11.59 | f         | {"score": 5.51, "status": "fail"} | Cillum Ullamco Non Cupidatat Occaecat Excepteur Incididunt Deserunt Sunt Commodo Anim Nulla Laborum Et Enim Consectetur Sed Ipsum Mollit Exercitation Voluptate Adipiscing Elit, Nostrud Esse Lorem Amet, Reprehenderit Ut Minim Magna Est Consequat. Tempor Ni | 2024-12-31 00:00:00
  3 | user_3   | user_3@mail.ru  |  58 |  67.99 | t         | {"score": 2.15, "status": "ok"}   | Eiusmod Ut Reprehenderit Ipsum Laboris Cupidatat In Minim Pariatur. Ea Dolor Esse Excepteur Occaecat Sint Aliqua. Quis Proident, Sed Commodo Est Incididunt Aliquip Ex Eu Labore Magna Consequat. Officia Anim Ad Nisi Veniam, Sunt Duis Nostrud Culpa Lorem Ad | 2025-06-26 00:00:00
  4 | user_4   | user_4@mail.ru  |  59 |  47.35 | t         | {"score": 5.28, "status": "ok"}   | Ut Est Commodo Ex Fugiat Adipiscing Velit Aliquip Duis Deserunt Ipsum Dolore Nulla Ea Non Lorem Qui Et Nisi Ad Ut Eiusmod Anim Sit Incididunt Enim Consectetur Minim Sint Ullamco Sed Aute Magna In Do Labore Quis Proident, Pariatur. Tempor Esse Occaecat Sun | 2025-01-30 00:00:00
  5 | user_5   | user_5@mail.ru  |  33 |  54.73 | t         | {"score": 0.26, "status": "ok"}   | Ipsum Ut In Ex Laborum Velit Qui Mollit Irure Commodo Ad Magna Excepteur Reprehenderit Minim Deserunt Enim Esse Ea Incididunt Fugiat Aliqua. Sed Ullamco Do Ut Occaecat Officia Quis Sit Nisi Consequat. Exercitation Nulla Tempor Elit, Labore Est Eu Dolor Co | 2025-03-18 00:00:00
  6 | user_6   | user_6@mail.ru  |  47 |  17.84 | t         | {"score": 8.36, "status": "ok"}   | Velit Fugiat Esse Ut Ea In Aliqua. Qui Ullamco Dolore Commodo Aute Eu Sint Aliquip Consequat. Ad Mollit Proident, Elit, Cupidatat Pariatur. Eiusmod Irure Culpa Do Adipiscing Tempor Non Est Nostrud Incididunt Et Consectetur Ex Id Veniam, Voluptate Magna La | 2024-11-04 00:00:00
  7 | user_7   | user_7@mail.ru  |  33 |   2.26 | t         | {"score": 5.20, "status": "fail"} | Nulla Ea Consectetur Et Eu Ut Velit Sunt Lorem In Nisi Voluptate Nostrud Pariatur. Reprehenderit Qui Cillum Incididunt Occaecat Labore Do Ut Laborum Eiusmod Ipsum Ex Fugiat Sed Esse Consequat. Magna Id Est Enim Excepteur Ad Non Sit Adipiscing Veniam, Exer | 2024-05-21 00:00:00
  8 | user_8   | user_8@mail.ru  |  27 |  77.49 | t         | {"score": 2.50, "status": "fail"} | Ipsum Voluptate Incididunt Proident, Eiusmod Dolor Magna Reprehenderit Deserunt Enim Tempor Consectetur Et Sed Do Est Consequat. Labore Lorem Excepteur Aliqua. Elit, Occaecat Ad Irure Culpa Amet, Dolore Aliquip Aute Eu Velit Sit Pariatur. Ut Exercitation  | 2024-01-10 00:00:00
  9 | user_9   | user_9@mail.ru  |  38 |  17.99 | t         | {"score": 6.32, "status": "fail"} | Ad Eiusmod Labore Mollit Nostrud Sunt Pariatur. Proident, Excepteur Amet, In Consequat. Non Ex Ea Nulla Laborum Sit Aute Eu Voluptate Dolor Aliqua. Adipiscing Deserunt Ipsum Qui Ut Culpa Dolore Magna Consectetur Cupidatat Aliquip Laboris Incididunt Enim D | 2024-09-08 00:00:00
 10 | user_10  | user_10@mail.ru |  31 |  83.61 | f         | {"score": 7.52, "status": "fail"} | Id Consequat. Do Exercitation Velit Culpa Sunt Anim Quis Ipsum Magna Aliquip Occaecat Mollit Reprehenderit Irure Ex Officia Sint Nostrud Laboris Est Laborum Nisi Dolore Proident, Excepteur In Cupidatat Aute Qui Labore Lorem Et Eiusmod Nulla Pariatur. Sit  | 2024-03-06 00:00:00
(10 rows)
```
