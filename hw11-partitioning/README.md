# Секционирование таблицы

Цели:

* научиться выполнять секционирование таблиц в PostgreSQL;
* повысить производительность запросов и упростив управление данными;

# Подготовка к решению домашнего задания

## Разворачивание базы данных полетов

Для решения домашнего задания предлагается использовать подготовленную демонстрационную базу данных с сайта
https://postgrespro.ru/docs/postgrespro/17/demodb-bookings.

Перейдите в раздел [Установка](https://postgrespro.ru/docs/postgrespro/17/demodb-bookings-installation) и скопируйте ссылку на ту версию
БД, которой хотите пользоваться.

Зайдите на свой сервер PostgreSQL через SSH, скачайте и распакуйте демо БД.

```shell
sudo su - postgres && cd ~/
wget https://edu.postgrespro.ru/demo-big.zip
unzip demo-big.zip
```

Далее разверните ее на сервере

```sql
psql -f demo-big-20170815.sql
```

## Проверка данных

После разворачивания БД подключитесь к серверу через psql или pgAdmin4 и проверьте данные.

```sql
SELECT * FROM bookings.aircrafts_data limit 10;
SELECT * FROM bookings.airports_data limit 10;
SELECT * FROM bookings.boarding_passes limit 10;
SELECT * FROM bookings.bookings limit 10;
SELECT * FROM bookings.flights limit 10;
SELECT * FROM bookings.seats limit 10;
SELECT * FROM bookings.ticket_flights limit 10;
SELECT * FROM bookings.tickets limit 10;
```

# Решение домашнего задания

## Анализ структуры данных

Для начала выделим самые большие таблицы с помощью запроса ниже и оценим, как можно их секционировать.

```sql
SELECT nspname || '.' || relname AS relation,
       pg_size_pretty(pg_total_relation_size(C.oid)) AS total_size
FROM pg_class C
LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
WHERE nspname NOT IN ('pg_catalog', 'information_schema')
  AND C.relkind <> 'i'
  AND nspname !~ '^pg_toast'
ORDER BY pg_total_relation_size(C.oid) DESC
LIMIT 20;
```

```
            relation            | total_size 
--------------------------------+------------
 bookings.boarding_passes       | 1102 MB
 bookings.ticket_flights        | 872 MB
 bookings.tickets               | 475 MB
 bookings.bookings              | 151 MB
 bookings.flights               | 32 MB
 bookings.seats                 | 144 kB
 bookings.airports_data         | 72 kB
 bookings.aircrafts_data        | 32 kB
 bookings.flights_flight_id_seq | 8192 bytes
```

### Анализ boarding_passes

Таблица содержит 7 925 812 записей с посадочными талонами.

```
   ticket_no   | flight_id | boarding_no | seat_no 
---------------+-----------+-------------+---------
 0005435189093 |    198393 |           1 | 27G
 0005435189119 |    198393 |           2 | 2D
 0005435189096 |    198393 |           3 | 18E
 0005435189117 |    198393 |           4 | 31B
 0005432208788 |    198393 |           5 | 28C
 0005435189151 |    198393 |           6 | 32A
 0005433655456 |    198393 |           7 | 31J
 0005435189129 |    198393 |           8 | 30C
 0005435629876 |    198393 |           9 | 30E
 0005435189100 |    198393 |          10 | 30F
...
```

Попробуем сгруппировать их по рядам.

```sql
SELECT
    RIGHT(seat_no, 1) AS seat_letter,
    COUNT(*) AS seat_count
FROM bookings.boarding_passes
GROUP BY seat_letter
ORDER BY seat_letter;
```

```
 seat_letter | seat_count 
-------------+-------------
 A           |    1 504 162
 B           |      835 726
 C           |    1 321 081
 D           |    1 422 644
 E           |      981 782
 F           |    1 143 056
 G           |      260 482
 H           |      257 899
 J           |       77 199
 K           |      121 781
(10 rows)
```

Распределение не самое оптимальное, а главное — такое разделение вряд ли удовлетворит запросы бизнеса. Вероятно, тем, кто анализирует
данные, будет неинтересно смотреть на загрузку рядов A, B, C и так далее. Гораздо важнее понять, какие места заняты на конкретном рейсе,
что отражает поле `flight_id`. Поэтому следует выполнять группировку по диапазону из 100 000 рейсов.

```sql
SELECT 
   (flight_id / 100000) AS flight_id_group,
   COUNT(*) AS count
FROM bookings.boarding_passes
GROUP BY (flight_id / 100000)
ORDER BY flight_id_group;
```

```
 flight_id_group |  count  
-----------------+---------
               0 | 4 806 292
               1 | 2 880 466
               2 |   239 054
(3 rows)
```

Уже намного лучше, хоть и есть периоды времени, когда билеты покупали мало, поэтому при том же количестве рейсов на них меньше
посадочных талонов. Остановимся пока на этом варианте, и перейдем ко второй по размеру таблице.

### Анализ ticket_flights

Таблица содержит 8 391 852 записей с категорией билета и его стоимость.

```
   ticket_no   | flight_id | fare_conditions |  amount   
---------------+-----------+-----------------+-----------
 0005432079221 |     36094 | Business        |  99800.00
 0005434861552 |     65405 | Business        |  49700.00
 0005432003235 |     89752 | Business        |  99800.00
 0005433567794 |    164215 | Business        | 105900.00
 0005432003470 |     89913 | Business        |  99800.00
 0005435834642 |    117026 | Business        | 199300.00
 0005432003656 |     90106 | Business        |  99800.00
 0005432949087 |    164161 | Business        | 105900.00
 0005432801137 |      9563 | Business        | 150400.00
 0005433557112 |    164098 | Business        | 105900.00
(10 rows)
...
```

Следуя той же логике, что и в предыдущем разборе, проведем группировку по `flight_id`.

```sql
SELECT 
  (flight_id / 100000) AS flight_id_group,
  COUNT(*) AS count
FROM bookings.ticket_flights
GROUP BY (flight_id / 100000)
ORDER BY flight_id_group;
```

```
 flight_id_group |  count  
-----------------+---------
               0 | 5 079 332
               1 | 3 058 699
               2 |   253 821
(3 rows)
```

Цифры сопоставимы с предыдущей таблицей, что вполне нас устраивают.

### Анализ tickets

Таблица содержит 2 949 857 записей с билетами и информацией о пассажирах.

```
   ticket_no   | book_ref | passenger_id |    passenger_name    |                                 contact_data                                 
---------------+----------+--------------+----------------------+------------------------------------------------------------------------------
 0005432000284 | 1A40A1   | 4030 855525  | MIKHAIL SEMENOV      | {"phone": "+70110137563"}
 0005432000285 | 13736D   | 8360 311602  | ELENA ZAKHAROVA      | {"phone": "+70670013989"}
 0005432000286 | DC89BC   | 4510 377533  | ILYA PAVLOV          | {"phone": "+70624013335"}
 0005432000287 | CDE08B   | 5952 253588  | ELENA BELOVA         | {"email": "e.belova.07121974@postgrespro.ru", "phone": "+70340423946"}
 0005432000288 | BEFB90   | 4313 788533  | VYACHESLAV IVANOV    | {"email": "vyacheslav-ivanov051968@postgrespro.ru", "phone": "+70417078841"}
 0005432000289 | A903E4   | 2742 028983  | NATALIYA NESTEROVA   | {"phone": "+70031478265"}
 0005432000290 | CC77B6   | 9873 744760  | ALEKSANDRA ARKHIPOVA | {"email": "arkhipovaa-1980@postgrespro.ru", "phone": "+70185914840"}
 0005432000291 | D530F6   | 2695 977692  | EVGENIY SERGEEV      | {"phone": "+70007395677"}
 0005432000292 | F26006   | 2512 253082  | TATYANA ZHUKOVA      | {"email": "zhukova_tatyana_121964@postgrespro.ru", "phone": "+70505293692"}
 0005432000293 | 739B4E   | 5763 638275  | ILYA KRASNOV         | {"email": "ilya_krasnov_081985@postgrespro.ru", "phone": "+70669365996"}
(10 rows)
```

Самое логичное разбиение тут — по идентификатору билета. Идентификатор представляет собой текстовое поле из 13 символов, которое по
международным стандартам состоит из:

* 000 — код авиакомпании,
* 5432000284 — уникальный номер самого билета.

Таким образом у каждой авиакомпании есть возможность реализовать 10 млрд. билетов (при условии если первый будет иметь №0000000000). А
для нас эта информация дает понимание, что мы можем секционировать эту таблицу по следующим критериям:

1. Если разделить это на 10 секций, то управлять ими будет затруднительно, так как каждая секция будет содержать 1 млрд. записей, что
   повлияет на скорость запросов.
2. Если разделить это на 100 секций, то управлять ими будет просто, а каждая секция будет содержать 100 млн. записей, что при наличии
   индекса на `ticket_no`, должно обеспечить нормальную скорость запросов.
3. Возможно стоит рассмотреть 200, 500 или 1000 секций, но для этого надо оценивать производительность каждого случая.

Проводим группировки по второму варианту с помощью запроса.

```sql
SELECT 
	LEFT(ticket_no, 6) as ticket_no,
	COUNT(*) as count
FROM bookings.tickets
GROUP BY LEFT(ticket_no, 6)
ORDER BY LEFT(ticket_no, 6);
```

```
 ticket_no |  count  
-----------+---------
 000543    | 2949857
 ```

### Анализ bookings

Последняя таблица для анализа содержит 2 111 110 записей с бронированием билетов.

```
 book_ref |       book_date        | total_amount 
----------+------------------------+--------------
 000004   | 2016-08-13 12:40:00+00 |     55800.00
 00000F   | 2017-07-05 00:12:00+00 |    265700.00
 000010   | 2017-01-08 16:45:00+00 |     50900.00
 000012   | 2017-07-14 06:02:00+00 |     37900.00
 000026   | 2016-08-30 08:08:00+00 |     95600.00
 00002D   | 2017-05-20 15:45:00+00 |    114700.00
 000034   | 2016-08-08 02:46:00+00 |     49100.00
 00003F   | 2016-12-12 12:02:00+00 |    109800.00
 000048   | 2016-09-16 22:57:00+00 |     92400.00
 00004A   | 2016-10-13 18:57:00+00 |     29000.00
(10 rows)
```

Один из вариантов секционирования будет по кварталам на основе дате бронирования.

```sql
SELECT
  EXTRACT(YEAR FROM bookings.book_date) AS year,
  EXTRACT(QUARTER FROM bookings.book_date) AS quarter,
  COUNT(*) AS count_records
FROM bookings.bookings
GROUP BY year, quarter
ORDER BY year, quarter;
```

```
 year | quarter | count_records 
------+---------+---------------
 2016 |       3 |        345 283
 2016 |       4 |        507 652
 2017 |       1 |        497 064
 2017 |       2 |        501 650
 2017 |       3 |        259 461
(5 rows)
```

Получилось достаточно мало записей в каждой секции, попробуем по годам.

```sql
SELECT
  EXTRACT(YEAR FROM bookings.book_date) AS year,
  COUNT(*) AS count_records
FROM bookings.bookings
GROUP BY year
ORDER BY year;
```

```
 year | count_records 
------+---------------
 2016 |        852935
 2017 |       1258175
(2 rows)
```

Остановимся на этом варианте.

## Определение типа секционирования

Из представленного анализа видно, что выбранные таблицы являются самыми большими и высоко нагруженными, поэтому проведем секционирование
каждой из них, при этом:

1. `bookings` секционируем с помощью `RANGE`, так как ключ секционирования это дата
2. `tickets` секционируем с помощью `HASH`, так как ключ секционирования это строка
3. `ticket_flights` секционируем с помощью `LIST` по полю `fare_conditions`, чисто для эксперимента
4. `boarding_passes` секционируем с помощью простого `RANGE`, так как ключ секционирования это число

Секционирование будем проводить в другой схеме, поэтому создадим ее предварительно для всех таблиц.

```sql
DROP SCHEMA IF EXISTS bookings_copy CASCADE;
CREATE SCHEMA bookings_copy;
```

## Секционирование таблиц

### Секционирование bookings

Создаем секционированную таблицу, скрипт создания получен через pgAdmin4. Так как секционирование идет по полю `book_date` — оно было
добавлено как PRIMARY KEY.

```sql
CREATE TABLE IF NOT EXISTS bookings_copy.bookings
(
    book_ref character(6) COLLATE pg_catalog."default" NOT NULL,
    book_date timestamp with time zone NOT NULL,
    total_amount numeric(10,2) NOT NULL,
    CONSTRAINT bookings_pkey PRIMARY KEY (book_ref, book_date)
) PARTITION BY RANGE (book_date);

CREATE TABLE bookings_copy.bookings_2016 PARTITION OF bookings_copy.bookings
	FOR VALUES FROM ('2016-01-01') TO ('2017-01-01');
CREATE TABLE bookings_copy.bookings_2017 PARTITION OF bookings_copy.bookings
	FOR VALUES FROM ('2017-01-01') TO ('2018-01-01');

 -- 0min 3sec
INSERT INTO bookings_copy.bookings
	OVERRIDING SYSTEM VALUE
	SELECT * FROM bookings.bookings;

VACUUM ANALYZE bookings_copy.bookings;
```

### Секционирование tickets

Данная таблица создается второй с одним важным изменением.

Дело в том, что в исходной БД таблица `tickets` ссылается на `bookings`
посредством FOREIGN KEY, но в PostgreSQL существует важное ограничение: внешние ключи не поддерживаются между секционированными таблицами,
если ключ секционирования отличается или если связи не выстроены строго по ключам секционирования.

В нашем случае:

* Таблица `bookings` секционирована по диапазону по полю `book_date`.
* Таблица `tickets` по хешу по полю `ticket_no`.
* Внешний ключ надо сделать по полю `book_ref`, которого в ключе секционирования `bookings` нет, а в `tickets` нет поля с ключом
  секционирования `book_date`.

Из-за этого прямой внешний ключ поставить нельзя — это архитектурное ограничение PostgreSQL. Поэтому создаем секционированную таблицу
без внешнего ключа и наполняем данными.

```sql
CREATE TABLE IF NOT EXISTS bookings_copy.tickets
(
    ticket_no character(13) COLLATE pg_catalog."default" NOT NULL,
    book_ref character(6) COLLATE pg_catalog."default" NOT NULL,
    passenger_id character varying(20) COLLATE pg_catalog."default" NOT NULL,
    passenger_name text COLLATE pg_catalog."default" NOT NULL,
    contact_data jsonb,
    CONSTRAINT tickets_pkey PRIMARY KEY (ticket_no)
    -- ,
    -- CONSTRAINT tickets_book_ref_fkey FOREIGN KEY (book_ref)
    --     REFERENCES bookings_copy.bookings (book_ref) MATCH SIMPLE
    --     ON UPDATE NO ACTION
    --     ON DELETE NO ACTION
) PARTITION BY HASH (ticket_no);

CREATE TABLE bookings_copy.tickets_0 PARTITION OF bookings_copy.tickets
   FOR VALUES WITH (MODULUS 10, REMAINDER 0);
CREATE TABLE bookings_copy.tickets_1 PARTITION OF bookings_copy.tickets
   FOR VALUES WITH (MODULUS 10, REMAINDER 1);
CREATE TABLE bookings_copy.tickets_2 PARTITION OF bookings_copy.tickets
   FOR VALUES WITH (MODULUS 10, REMAINDER 2);
CREATE TABLE bookings_copy.tickets_3 PARTITION OF bookings_copy.tickets
   FOR VALUES WITH (MODULUS 10, REMAINDER 3);
CREATE TABLE bookings_copy.tickets_4 PARTITION OF bookings_copy.tickets
   FOR VALUES WITH (MODULUS 10, REMAINDER 4);
CREATE TABLE bookings_copy.tickets_5 PARTITION OF bookings_copy.tickets
   FOR VALUES WITH (MODULUS 10, REMAINDER 5);
CREATE TABLE bookings_copy.tickets_6 PARTITION OF bookings_copy.tickets
   FOR VALUES WITH (MODULUS 10, REMAINDER 6);
CREATE TABLE bookings_copy.tickets_7 PARTITION OF bookings_copy.tickets
   FOR VALUES WITH (MODULUS 10, REMAINDER 7);
CREATE TABLE bookings_copy.tickets_8 PARTITION OF bookings_copy.tickets
   FOR VALUES WITH (MODULUS 10, REMAINDER 8);
CREATE TABLE bookings_copy.tickets_9 PARTITION OF bookings_copy.tickets
   FOR VALUES WITH (MODULUS 10, REMAINDER 9);

 -- 0min 18sec
INSERT INTO bookings_copy.tickets
	OVERRIDING SYSTEM VALUE
	SELECT * FROM bookings.tickets;

VACUUM ANALYZE bookings_copy.tickets;
```

#### Как строить архитектуру с секционированием и связью?

##### Вариант 1: Не создавать внешний ключ в БД, а контролировать целостность на уровне приложения

* Это самый простой вариант, если архитектура сложная.
* Секционируете обе таблицы независимо, без внешних ключей.
* Логику связи и проверки целостности выполняете в приложении.

##### Вариант 2: Использовать одинаковое секционирование и ключи, чтобы внешний ключ был валиден

* Секционировать и `bookings`, и `tickets` одинаково, например, по `book_ref` и `book_date` (составной ключ).
* В таблице `tickets` добавить поля `book_ref` и `book_date` и сделать составной внешний ключ на `(book_ref, book_date)`.

##### Вариант 3: Создать денормализованное поле в tickets с нужным ключом секционирования

* Добавить поле `book_date` в `tickets` и поддерживать его через триггер вставки/обновления с копированием из `bookings`.
* Тогда можно сделать одинаковое секционирование и внешний ключ.
* Минус — дублирование данных, возможны рассинхронизации.

#### ЧТО ДЕЛАТЬ?!

Ничего. В жизни мы бы использовали либо секционирование для `bookings` по первичному ключу, либо пошли по **Варианту 1**. А в случае с
домашним заданием нам:

1. интересно провести секционирование по дате
2. интересно попасть в подобную ситуацию, чтобы начать в ней разбираться

### Секционирование ticket_flights

Создаем следующую секционированную таблицу. Обратите внимание на внешние ключи.

```sql
CREATE TABLE IF NOT EXISTS bookings_copy.ticket_flights
(
    ticket_no character(13) COLLATE pg_catalog."default" NOT NULL,
    flight_id integer NOT NULL,
    fare_conditions character varying(10) COLLATE pg_catalog."default" NOT NULL,
    amount numeric(10,2) NOT NULL,
    CONSTRAINT ticket_flights_pkey PRIMARY KEY (ticket_no, flight_id, fare_conditions),
    CONSTRAINT ticket_flights_flight_id_fkey FOREIGN KEY (flight_id)
        REFERENCES bookings_copy.flights (flight_id) MATCH SIMPLE
        ON UPDATE NO ACTION
        ON DELETE NO ACTION,
    CONSTRAINT ticket_flights_ticket_no_fkey FOREIGN KEY (ticket_no)
        REFERENCES bookings_copy.tickets (ticket_no) MATCH SIMPLE
        ON UPDATE NO ACTION
        ON DELETE NO ACTION,
    CONSTRAINT ticket_flights_amount_check CHECK (amount >= 0::numeric),
    CONSTRAINT ticket_flights_fare_conditions_check CHECK (fare_conditions::text = ANY (ARRAY['Economy'::character varying::text, 'Comfort'::character varying::text, 'Business'::character varying::text]))
) PARTITION BY LIST (fare_conditions);

CREATE TABLE bookings_copy.ticket_flights_e PARTITION OF bookings_copy.ticket_flights
	FOR VALUES IN ('Economy');
CREATE TABLE bookings_copy.ticket_flights_c PARTITION OF bookings_copy.ticket_flights
	FOR VALUES IN ('Comfort');
CREATE TABLE bookings_copy.ticket_flights_b PARTITION OF bookings_copy.ticket_flights
	FOR VALUES IN ('Business');
	
 -- 2min 20sec
INSERT INTO bookings_copy.ticket_flights
	OVERRIDING SYSTEM VALUE
	SELECT * FROM bookings.ticket_flights;

VACUUM ANALYZE bookings_copy.ticket_flights;
```

### Секционирование boarding_passes

И последняя секционированная таблица на сегодня.

```sql
CREATE TABLE IF NOT EXISTS bookings_copy.boarding_passes
(
    ticket_no character(13) COLLATE pg_catalog."default" NOT NULL,
    flight_id integer NOT NULL,
    boarding_no integer NOT NULL,
    seat_no character varying(4) COLLATE pg_catalog."default" NOT NULL,
	CONSTRAINT boarding_passes_pkey PRIMARY KEY (ticket_no, flight_id),
	CONSTRAINT boarding_passes_flight_id_boarding_no_key UNIQUE (flight_id, boarding_no),
	CONSTRAINT boarding_passes_flight_id_seat_no_key UNIQUE (flight_id, seat_no),
	CONSTRAINT boarding_passes_ticket_no_fkey FOREIGN KEY (ticket_no, flight_id)
        REFERENCES bookings.ticket_flights (ticket_no, flight_id) MATCH SIMPLE
		ON UPDATE NO ACTION
        ON DELETE NO ACTION
) PARTITION BY RANGE (flight_id);

CREATE TABLE bookings_copy.boarding_passes_0 PARTITION OF bookings_copy.boarding_passes
	FOR VALUES FROM (0) TO (100000);
CREATE TABLE bookings_copy.boarding_passes_1 PARTITION OF bookings_copy.boarding_passes
	FOR VALUES FROM (100000) TO (200000);
CREATE TABLE bookings_copy.boarding_passes_2 PARTITION OF bookings_copy.boarding_passes
	FOR VALUES FROM (200000) TO (300000);
CREATE TABLE bookings_copy.boarding_passes_3 PARTITION OF bookings_copy.boarding_passes
	FOR VALUES FROM (300000) TO (400000);

 -- 1min 12sec
INSERT INTO bookings_copy.boarding_passes
	OVERRIDING SYSTEM VALUE
	SELECT * FROM bookings.boarding_passes;

VACUUM ANALYZE bookings_copy.boarding_passes;
```

## Сравнение результатов

### Таблица bookings

#### Полная выборка

Секционирование не дает преимущества при полной выборке так как приходится обходить каждую секцию.

```sql
EXPLAIN ANALYZE SELECT * FROM bookings.bookings;
/*
 Seq Scan on bookings  (cost=0.00..34599.10 rows=2111110 width=21) (actual time=0.025..89.084 rows=2111110 loops=1)
 Planning Time: 0.100 ms
 Execution Time: 136.449 ms
*/

EXPLAIN ANALYZE SELECT * FROM bookings_copy.bookings;
/*
 Append  (cost=0.00..45113.65 rows=2111110 width=21) (actual time=0.020..162.218 rows=2111110 loops=1)
   ->  Seq Scan on bookings_2016 bookings_1  (cost=0.00..13962.35 rows=852935 width=21) (actual time=0.020..45.309 rows=852935 loops=1)
   ->  Seq Scan on bookings_2017 bookings_2  (cost=0.00..20595.75 rows=1258175 width=21) (actual time=0.005..37.936 rows=1258175 loops=1)
 Planning Time: 0.128 ms
 Execution Time: 203.253 ms
*/
```

#### Выборка по ключу

Секционирование не дает преимущества при выборке по ключу, так как требуется обход индексов в каждой секции.

```sql
EXPLAIN SELECT * FROM bookings.bookings WHERE book_ref = '000004';
/*
 Index Scan using bookings_pkey on bookings  (cost=0.43..2.65 rows=1 width=21) (actual time=0.461..0.465 rows=1 loops=1)
 Planning Time: 0.140 ms
 Execution Time: 0.501 m
*/

EXPLAIN SELECT * FROM bookings_copy.bookings WHERE book_ref = '000004';
/*
 Append  (cost=0.42..5.30 rows=2 width=21) (actual time=0.182..0.737 rows=1 loops=1)
   ->  Index Scan using bookings_2016_pkey on bookings_2016 bookings_1  (cost=0.42..2.64 rows=1 width=21) (actual time=0.181..0.183 
rows=1 loops=1)
   ->  Index Scan using bookings_2017_pkey on bookings_2017 bookings_2  (cost=0.43..2.65 rows=1 width=21) (actual time=0.547..0.547 
rows=0 loops=1)
 Planning Time: 0.232 ms
 Execution Time: 0.792 ms
*/
```

#### Выборка по дате

Секционирование дает небольшое преимущество при выборке по дате, так как происходит сканирование одной секции с меньшим числом строк 
данных. Однако если 

```sql
EXPLAIN ANALYZE SELECT * FROM bookings.bookings WHERE book_date = '2016-08-13';
/*
 Gather  (cost=1000.00..25483.86 rows=5 width=21) (actual time=34.728..53.054 rows=3 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on bookings  (cost=0.00..24483.36 rows=2 width=21) (actual time=31.036..46.696 rows=1 loops=3)
         Filter: (book_date = '2016-08-13 00:00:00+00'::timestamp with time zone)
         Rows Removed by Filter: 703702
 Planning Time: 0.101 ms
 Execution Time: 53.079 ms
*/

EXPLAIN ANALYZE SELECT * FROM bookings_copy.bookings WHERE book_date = '2016-08-13';
/*
 Index Scan using bookings_2016_pkey on bookings_2016 bookings  (cost=0.42..10010.71 rows=5 width=21) (actual time=12.740..35.954 
rows=3 loops=1)
   Index Cond: (book_date = '2016-08-13 00:00:00+00'::timestamp with time zone)
 Planning Time: 0.204 ms
 Execution Time: 36.002 ms
*/
```

Однако это преимущество теряется при создании индексов на исходной таблице.

```sql
CREATE INDEX ON bookings.bookings (book_date);
ANALYZE bookings.bookings;
EXPLAIN ANALYZE SELECT * FROM bookings.bookings WHERE book_date = '2016-08-13';
/*
 Index Scan using bookings_book_date_idx on bookings  (cost=0.43..7.12 rows=5 width=21) (actual time=0.028..0.035 rows=3 loops=1)
   Index Cond: (book_date = '2016-08-13 00:00:00+00'::timestamp with time zone)
 Planning Time: 0.076 ms
 Execution Time: 0.051 ms
*/

CREATE INDEX ON bookings_copy.bookings (book_date);
ANALYZE bookings_copy.bookings;
EXPLAIN ANALYZE SELECT * FROM bookings_copy.bookings WHERE book_date = '2016-08-13';
/*
 Index Scan using bookings_2016_book_date_idx on bookings_2016 bookings  (cost=0.42..7.11 rows=5 width=21) (actual time=0.047..0.059 rows=3 loops=1)
   Index Cond: (book_date = '2016-08-13 00:00:00+00'::timestamp with time zone)
 Planning Time: 0.664 ms
 Execution Time: 0.086 ms
*/
```

#### Выборка по диапазону дат

И только при выборке по диапазону дат мы видим существенное повышение скорости выполнения запроса.

```sql
EXPLAIN ANALYZE SELECT * FROM bookings.bookings WHERE book_date > '2016-08-13' AND book_date < '2017-01-01';
/*
 Index Scan using bookings_book_date_idx1 on bookings  (cost=0.43..31908.75 rows=784167 width=21) (actual time=0.017..283.488 rows=777983 loops=1)
   Index Cond: ((book_date > '2016-08-13 00:00:00+00'::timestamp with time zone) AND (book_date < '2017-01-01 00:00:00+00'::timestamp with time zone))
 Planning Time: 0.049 ms
 Execution Time: 296.656 ms
*/

EXPLAIN ANALYZE SELECT * FROM bookings_copy.bookings WHERE book_date > '2016-08-13' AND book_date < '2017-01-01';
/*
 Seq Scan on bookings_2016 bookings  (cost=0.00..18227.03 rows=778765 width=21) (actual time=0.005..37.297 rows=777983 loops=1)
   Filter: ((book_date > '2016-08-13 00:00:00+00'::timestamp with time zone) AND (book_date < '2017-01-01 00:00:00+00'::timestamp with time zone))
   Rows Removed by Filter: 74952
 Planning Time: 0.136 ms
 Execution Time: 51.432 ms
*/
```

