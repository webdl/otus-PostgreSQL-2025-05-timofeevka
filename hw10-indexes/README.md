# Работа с индексами

Цели:

* знать и уметь применять основные виды индексов PostgreSQL
* строить и анализировать план выполнения запроса
* уметь оптимизировать запросы для с использованием индексов

# Подготовка к решению домашнего задания

## Функция генерации текста

Для работы с GIN индексом используем простую функцию генерации lorem ipsum текста.

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

Наполним ее 1,000,000 сгенерированными данными (генерация занимает ~45 сек):

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
FROM generate_series(1, 1000000) gs;
```

Пример полученных записей:

```
 id | username |     email      | age | rating | is_active |               meta                |                                                                                                                              about                                                                                                                              |     created_at      
----+----------+----------------+-----+--------+-----------+-----------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+---------------------
  1 | user1    | user1@mail.ru  |  22 |  56.94 | t         | {"score": 5.91, "status": "fail"} | Quis Duis Labore Eiusmod Eu Nulla Consectetur Occaecat Nisi Aliqua. Cupidatat Reprehenderit Do Esse Sit Et Fugiat Voluptate Est Irure Culpa Minim Ex Aute In Ut Excepteur Dolore Dolor Deserunt Enim Ipsum Officia Magna Ad Anim Incididunt Proident, Tempor Su | 2024-10-20 00:00:00
  2 | user2    | user2@mail.ru  |  36 |  50.30 | t         | {"score": 2.81, "status": "ok"}   | Aute Incididunt Deserunt Ad Eiusmod Culpa Velit Ipsum Ut Elit, Mollit Sint Sed Laboris Cillum Ullamco Consequat. Anim Lorem Enim Occaecat Magna Est Id Pariatur. Esse Dolore In Consectetur Nisi Nulla Ea Ut Proident, Sunt Veniam, Aliquip Minim Qui Quis Volu | 2024-10-05 00:00:00
  3 | user3    | user3@mail.ru  |  54 |  26.53 | f         | {"score": 7.80, "status": "fail"} | Nulla Officia Ipsum Consequat. Occaecat Fugiat Pariatur. Labore Laboris Lorem Sed Anim Tempor Incididunt Cupidatat Ut Do Sit Et Veniam, Excepteur Ea Minim Nostrud Quis Magna Duis In Elit, Ex Commodo Consectetur Nisi Eiusmod Exercitation Ut Deserunt Aliqui | 2025-05-07 00:00:00
  4 | user4    | user4@mail.ru  |  44 |  57.60 | t         | {"score": 2.14, "status": "ok"}   | Sint Excepteur Et Cupidatat Pariatur. Mollit Adipiscing Ea Voluptate Amet, Minim Id Aliquip Nulla Aliqua. Quis Ipsum Eiusmod Ad Do Irure Duis Proident, Cillum Tempor Labore Sit Est Ex Fugiat Eu Laborum Sunt Magna Dolore Veniam, Enim Lorem Consectetur Comm | 2024-01-20 00:00:00
  5 | user5    | user5@mail.ru  |  44 |  37.55 | t         | {"score": 3.12, "status": "fail"} | Anim Mollit Quis Aute Sint Sed Esse Ut Voluptate Aliquip Consequat. Veniam, Lorem Proident, Minim Dolore Nostrud Dolor Aliqua. Exercitation Velit Elit, Pariatur. Officia Nulla Labore Ex Incididunt Reprehenderit Est Et Adipiscing Laborum Commodo Eu Non Id  | 2025-03-22 00:00:00
  6 | user6    | user6@mail.ru  |  26 |  42.73 | t         | {"score": 1.18, "status": "ok"}   | Pariatur. Culpa Esse Duis Nulla Proident, Ea Eu Et Tempor Aliqua. Amet, Reprehenderit Occaecat Laborum Quis Ad Ut Lorem Consequat. Mollit Irure Magna Non Voluptate Sunt Dolor Ut Cillum Nostrud Aliquip Consectetur In Est Laboris Deserunt Nisi Do Aute Enim  | 2025-05-01 00:00:00
  7 | user7    | user7@mail.ru  |  25 |  22.96 | t         | {"score": 9.03, "status": "ok"}   | Pariatur. Fugiat Dolore Eu Est Aliqua. Officia Cupidatat Cillum Adipiscing Et Velit Qui Lorem Ullamco Sunt Id Deserunt Labore Duis Nostrud Aute Quis In Ad Sint Mollit Commodo Voluptate Ipsum Aliquip Ut Do Culpa Incididunt Esse Occaecat Non Anim Tempor Eiu | 2024-09-05 00:00:00
  8 | user8    | user8@mail.ru  |  20 |  69.09 | t         | {"score": 5.92, "status": "fail"} | Amet, In Dolor Id Consectetur Et Quis Cupidatat Officia Incididunt Est Ex Anim Tempor Fugiat Nulla Pariatur. Aute Sit Non Cillum Qui Adipiscing Aliqua. Nisi Commodo Occaecat Sed Proident, Esse Ullamco Consequat. Duis Sint Ut Culpa Dolore Magna Voluptate E | 2025-03-24 00:00:00
  9 | user9    | user9@mail.ru  |  24 |  18.34 | f         | {"score": 6.50, "status": "ok"}   | Cillum Officia Ut Veniam, Ut Esse Voluptate Magna Dolor Et Minim Ad Laboris Nisi In Id Reprehenderit Aliquip Tempor Mollit Enim Ipsum Non Velit Irure Culpa Elit, Ex Consequat. Occaecat Quis Duis Do Consectetur Ullamco Sed Aliqua. Qui Pariatur. Est Exercit | 2024-04-17 00:00:00
 10 | user10   | user10@mail.ru |  63 |   0.57 | t         | {"score": 3.54, "status": "ok"}   | Fugiat Consectetur Sed Exercitation Non Et Elit, Ea Ex Quis Velit Esse Laboris Sunt Amet, Mollit Eu Tempor Nisi Sint Minim Ut Duis Eiusmod Incididunt Ullamco Veniam, Aute Anim Deserunt Est Proident, Sit Voluptate In Magna Dolore Ad Occaecat Dolor Lorem Co | 2024-02-18 00:00:00
(10 rows)
```

## Размеры полученной БД и индексов

Далее по ходу выполнения ДЗ я буду замерять размеры полученных индексов, чтобы оценивать дополнительные затраты на ресурсы, так как
индексы не бесплатные. Измерения проводятся с помощью запроса ниже.

```sql
SELECT pg_size_pretty(pg_total_relation_size('idx_users_rating'));

 pg_size_pretty 
----------------
 567 MB
```

# Решение домашнего задания

Придумаем бизнес кейсы и попробуем добиться максимальной производительности поиска для них.

## Поиск пользователя по username

Предположим, что в нашей системе есть интерфейс, где можно найти пользователя по его username. Запрос с интерфейса выглядит следующим
образом:

```sql
EXPLAIN
SELECT username, email FROM users
	WHERE username = 'user_777';
```

Поиск с предикатом `=` выполняется быстро, так как индекс уже существует для работы `UNIQUE`.

```
 Index Scan using users_username_key on users  (cost=0.43..2.65 rows=1 width=32)
   Index Cond: ((username)::text = 'user_777'::text)
```

Однако, с точки зрения пользователя интерфейсом нас интересует поиск по подстроке, а текущий индекс не позволяет искать по предикату
`LIKE`. Поэтому запрос ниже не использует индекс, а выполняет сканирование по всем данным.

```sql
EXPLAIN
SELECT username, email FROM users
	WHERE username LIKE '%user777%';
```

```
 Gather  (cost=1000.00..55980.33 rows=100 width=28)
   Workers Planned: 2
   ->  Parallel Seq Scan on users  (cost=0.00..54970.33 rows=42 width=28)
         Filter: ((username)::text ~~ '%user777%'::text
```

Чтобы ускорить запросы добавляем GIN-индекс на поле и смотрим план предыдущего запроса.

```sql
-- Расширение позволяющее добавлять GIN-индекс на varchar поля
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE INDEX idx_users_username
	ON public.users USING gin 
	(username gin_trgm_ops);
```

```
 Bitmap Heap Scan on users  (cost=28.40..139.20 rows=100 width=28)
   Recheck Cond: ((username)::text ~~ 'user777%'::text)
   ->  Bitmap Index Scan on idx_users_username  (cost=0.00..28.38 rows=100 width=0)
         Index Cond: ((username)::text ~~ 'user777%'::text)
```

Таким образом затраты на поиск с предикатом `LIKE` сократились с 55980 до 139 условных единиц.

**Размер индекса:**

```sql
SELECT pg_size_pretty(pg_total_relation_size('idx_users_username'));

 pg_size_pretty 
----------------
 19 MB
```

## Поиск пользователей по рейтингу

Еще один бизнес-кейс — поиск пользователей с рейтингом более 90 единиц. Время выполнения которого без индексов составляет ~200мс.

```sql
EXPLAIN ANALYZE
SELECT id, username, email, rating
FROM users
WHERE rating >= 90.0;
```

```
 Seq Scan on users  (cost=0.00..62262.00 rows=101055 width=38) (actual time=0.036..203.760 rows=99409 loops=1)
   Filter: (rating >= 90.0)
   Rows Removed by Filter: 90059
```

Добавляем простой B-Tree индекс и видим, что время выполнения сократилось до ~65мс.

```sql
CREATE INDEX idx_users_rating
    ON users USING btree
    (rating);
```

```
 Bitmap Heap Scan on users  (cost=1089.40..52114.59 rows=101055 width=38) (actual time=30.942..68.496 rows=99409 loops=1)
   Recheck Cond: (rating >= 90.0)
   Heap Blocks: exact=43553
   ->  Bitmap Index Scan on idx_users_rating  (cost=0.00..1064.14 rows=101055 width=0) (actual time=24.802..24.802 rows=99409 loops=1)
         Index Cond: (rating >= 90.0)
```

**Размер индекса:**

```sql
SELECT pg_size_pretty(pg_total_relation_size('idx_users_rating'));

 pg_size_pretty 
----------------
 21 MB
```

### Оптимизация №1

Однако как видно из плана запроса выше — время на поиск по индексу составляет ~24мс, в то время как остальное время занимает получение
атрибутов запрос из данных. Так как получение пользователей с высоким рейтингом является частой операцией, то мы можем ускорить ее
работу путем добавления требуемых данных в индекс, что позволит не обращаться за ними в таблицу и значительно ускорить запрос.

```sql
CREATE INDEX idx_users_rating_with_data
    ON users USING btree
    (rating)
    INCLUDE(id, username, email);
```

```sql
 Index Only Scan using idx_users_rating_with_data on users  (cost=0.42..2676.39 rows=101055 width=38) (actual time=0.049..20.754 rows=99409 loops=1)
   Index Cond: (rating >= 90.0)
   Heap Fetches: 0
```

Но, как можно увидеть ниже, платой за ускорение запроса стали размеры индексов.

**Размер индекса:**

```sql
SELECT pg_size_pretty(pg_total_relation_size('idx_users_rating_with_data'));

 pg_size_pretty 
----------------
 64 MB
```

### Оптимизация №2

Попробуем оптимизировать индекс выше, чтобы он не занимал 11% всех данных в таблице users. Так как в нашем запросе мы ищем пользователей
с рейтингом только 90 и выше, то почему бы не создать **частичный индекс** только на эти данные?

```sql
DROP INDEX idx_users_rating_with_data;

CREATE INDEX idx_users_rating_with_data
    ON users USING btree
    (rating)
	INCLUDE(id, username, email)
	WHERE rating >= 90.0;
```

Проверяем, что производительность поиска не упала

```sql
EXPLAIN ANALYZE
SELECT id, username, email, rating
FROM users
WHERE rating >= 90.0;
```

```
 Index Only Scan using idx_users_rating_with_data on users  (cost=0.42..2402.31 rows=101055 width=38) (actual time=0.030..18.464 rows=99409 loops=1)
   Heap Fetches: 0
```

А размер индекса существенно сократился:

```sql
SELECT pg_size_pretty(pg_total_relation_size('idx_users_rating_with_data'));

 pg_size_pretty 
----------------
 6504 kB
```

Да! Всего одной строчкой мы сократили размер текущего индекс в 10 раз!

## Поиск чемпионов геймификации

В атрибуте `meta` хранится дополнительная информация о пользователе, такая как `score` и `status`, которые пользователь получает во 
время участия в геймификации на сервисе. Давайте найдем пользователей, которые имеют самый высоких бал и посмотрим план запроса.

```
SELECT id, username, meta
FROM users
WHERE is_active = true
	AND (meta->>'score')::numeric = 9.7
	AND meta->>'status' = 'ok';
```

```
 Gather  (cost=1000.00..61311.97 rows=1333 width=59) (actual time=0.338..105.217 rows=12320 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on users  (cost=0.00..60178.67 rows=555 width=59) (actual time=0.043..98.353 rows=4107 loops=3)
         Filter: (is_active AND ((meta ->> 'status'::text) = 'ok'::text) AND (((meta ->> 'score'::text))::numeric >= 9.7))
         Rows Removed by Filter: 329227
```

### Оптимизация №1

Достаточно неплохой результат в 105 мс. Но давайте ускорим его с помощью индексов. Для начала попробуем создать индексы для атрибутов 
`is_active` и `meta` по отдельности.

```sql
CREATE INDEX idx_users_is_active
    ON users USING btree
    (is_active);

CREATE INDEX idx_users_meta
    ON users USING gin
    (meta);
    
ANALYZE users;
```

```sql
EXPLAIN ANALYZE
SELECT id, username, meta
FROM users
WHERE is_active = true
	AND (meta->>'score')::numeric >= 9.7
	AND meta->>'status' = 'ok';
```

```
 Gather  (cost=1000.00..61311.13 rows=1324 width=59) (actual time=0.792..111.230 rows=12320 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on users  (cost=0.00..60178.73 rows=552 width=59) (actual time=0.096..103.714 rows=4107 loops=3)
         Filter: (is_active AND ((meta ->> 'status'::text) = 'ok'::text) AND (((meta ->> 'score'::text))::numeric >= 9.7))
         Rows Removed by Filter: 329227
```

### Оптимизация №2

Как видим планировщик решил их не использовать и посчитал, что быстрее будет выполнить обычный перебор всех записей. Удаляем индексы и 
пишем **составной индекс**.

```sql
DROP INDEX idx_users_is_active;
DROP INDEX idx_users_meta;
```

```sql
CREATE INDEX idx_users_is_active_and_meta
    ON users 
    (is_active, meta);

ANALYZE users;
```

Увы, но и это не помогло...

```
 Gather  (cost=1000.00..61311.67 rows=1330 width=59) (actual time=0.490..110.084 rows=12320 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on users  (cost=0.00..60178.67 rows=554 width=59) (actual time=0.074..102.329 rows=4107 loops=3)
         Filter: (is_active AND ((meta ->> 'status'::text) = 'ok'::text) AND (((meta ->> 'score'::text))::numeric >= 9.7))
         Rows Removed by Filter: 329227
```

### Оптимизация №3

Вспомним про **частичные индексы** и сделаем комбинацию из составного и частичного индекса. Но сначала давайте подробнее изучим, на 
основе каких данных планировщик будет решать, нужно ли использовать индекс для атрибута `is_active`.

```sql
SELECT
	most_common_vals AS mcv,
	left(most_common_freqs::text, 60) || '...' AS mcf
	FROM pg_stats
	WHERE tablename = 'users'
		AND attname = 'is_active' \gx
```

```
-[ RECORD 1 ]-------------------
mcv | {t,f}
mcf | {0.79906666,0.20093334}..
```

Как можно увидеть значение `is_active = true` порядка 80%, поэтому планировщику не имеет смысла использовать индекс при поиске, так как 
накладных расходов на работу с индексом будет больше, чем затрат на фильтрацию значения `false` в процессе перебора всех данных. Из 
этого мы делаем вывод, что создавать индекс на атрибут `is_active` для данного случая не требуется.

Создаем новый индекс и оцениваем результат.

```sql
CREATE INDEX idx_users_is_active_and_meta_top_score
    ON users 
    (is_active, meta)
	WHERE (meta->>'score')::numeric >= 9.7 AND meta->>'status' = 'ok';

ANALYZE users;
```

```sql
EXPLAIN ANALYZE
SELECT id, username, meta
FROM users
WHERE is_active = true
	AND (meta->>'score')::numeric >= 9.7
	AND meta->>'status' = 'ok';
```

```
 Index Scan using idx_users_is_active_and_meta_top_score on users  (cost=0.41..1124.79 rows=1330 width=59) (actual time=0.020..21.378 rows=12320 loops=1)
   Index Cond: (is_active = true)
```

Неплохо, мы сократили запрос в 5 раз! Обратите внимание, что сам индекс занимает незначительный объем на диске. 

**Размер индекса:**

```sql
SELECT pg_size_pretty(pg_total_relation_size('idx_users_is_active_and_meta_top_score'));

 pg_size_pretty 
----------------
 1312 kB
```

### Оптимизация №4

Вспоминаем наш предыдущий опыт и то, что после получения записей из индекса планировщик обращается на диск для извлечения самих данных. 
Оптимизируем наш предыдущий индекс, добавив нужные данные в него, и посмотрим на результат.

```sql
DROP INDEX idx_users_is_active_and_meta_top_score;

CREATE INDEX idx_users_is_active_and_meta_top_score
    ON users 
    (is_active, meta)
	INCLUDE(id, username)
	WHERE (meta->>'score')::numeric >= 9.7 AND meta->>'status' = 'ok';

ANALYZE users;
```

```
 Index Only Scan using idx_users_is_active_and_meta_top_score on users  (cost=0.41..40.24 rows=1333 width=59) (actual time=0.178..4.665 rows=12320 loops=1)
   Index Cond: (is_active = true)
   Heap Fetches: 0
```

В 20 раз быстрее относительного первого запроса! А что с размером?

**Размер индекса:**

```sql
SELECT pg_size_pretty(pg_total_relation_size('idx_users_is_active_and_meta_top_score'));

 pg_size_pretty 
----------------
 1312 kB
```
