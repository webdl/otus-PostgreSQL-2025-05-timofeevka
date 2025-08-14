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
что отражает поле `flight_id`. Поэтому следует выполнять группировку по рейсам, но не по каждому отдельно, а по диапазону из 25 000 рейсов.

```sql
SELECT 
  (flight_id / 25000) AS flight_id_group,
  COUNT(*) AS count
FROM bookings.boarding_passes
GROUP BY (flight_id / 25000)
ORDER BY flight_id_group;
```

```
 flight_id_group |  count  
-----------------+----------
               0 | 1 341 703
               1 | 1 267 194
               2 | 1 242 192
               3 |   955 203
               4 |   489 979
               5 |   656 126
               6 |   848 940
               7 |   885 421
               8 |   239 054
```

Уже намного лучше, хоть и есть периоды времени, когда билеты покупали очень мало. Остановимся пока на этом варианте, и перейдем ко второй 
по размеру таблице.

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
  (flight_id / 25000) AS flight_id_group,
  COUNT(*) AS count
FROM bookings.ticket_flights
GROUP BY (flight_id / 25000)
ORDER BY flight_id_group;
```

```
 flight_id_group |  count  
-----------------+----------
               0 | 1 415 626
               1 | 1 333 474
               2 | 1 315 833
               3 | 1 014 399
               4 |   519 349
               5 |   697 606
               6 |   901 978
               7 |   939 766
               8 |   253 821
(9 rows)
```

И увидим сопоставимые цифры, которые нас устраивают.

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

Таким образом можно провести группировки с помощью следующего запроса.

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
