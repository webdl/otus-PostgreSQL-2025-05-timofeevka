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

1. `boarding_passes` секционируем с помощью простого `RANGE`, так как ключ секционирования это число
2. `ticket_flights` секционируем с помощью `LIST` по полю `fare_conditions`, чисто для эксперимента
3. `tickets` секционируем с помощью `HASH`, так как ключ секционирования это строка
4. `bookings` секционируем с помощью `RANGE`, так как ключ секционирования это дата

