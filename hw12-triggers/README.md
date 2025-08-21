# Триггеры, поддержка заполнения витрин

Цели:

* Создать триггер для поддержки витрины в актуальном состоянии.

# Подготовка к решению домашнего задания

## Схема данных

Создадим таблицы товаров и продаж

```sql
DROP SCHEMA IF EXISTS pract_triggers CASCADE;
CREATE SCHEMA pract_triggers;

SET search_path = pract_triggers, publ

-- товары:
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);

-- Продажи
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);
```

Заполним их данными

```sql
INSERT INTO goods (goods_id, good_name, good_price)
    VALUES 	(1, 'Спички хозайственные', .50),
            (2, 'Автомобиль Ferrari FXX K', 185000000.01);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
```

Создадим отчет по проданным товарам

```sql
SELECT G.good_name, sum(G.good_price * S.sales_qty)
    FROM goods G
    INNER JOIN sales S ON S.good_id = G.goods_id
    GROUP BY G.good_name;
```

С увеличением объёма данных отчет стал создаваться медленно, поэтому принято решение денормализовать БД, создать таблицу

```sql
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);
```

## Бизнес требования

1. Заведение новой продажи отражается в отчете: появляется запись с именем товара и суммой, где сумма — это кол-во проданного товара
   умноженное на его стоимость
2. Удаление продажи приводит к удалению из отчета суммы по этой продаже
3. Изменение количества проданных товаров приводит к изменению в отчете суммы по этой продаже

## Функциональные требования

1. Таблица `good_sum_mart` заполнена с помощью разового SQL запроса (можно использовать запрос текущего отчета)
2. Для таблицы `sales` создан триггер, который:
    1. Срабатывает на событиях `INSERT`, `UPDATE`, `DELETE`
    2. Срабатывает после модификации таблицы `sales` (`AFTER`)
3. При событии `INSERT` триггер:
    1. Считает сумму по формуле `goods.good_price * sales.sales_qty`
    2. Производит поиск существующих записей в `good_sum_mart`, и:
        1. Если запись не найдена — производит вставку новой записи
        2. Если запись найдена — добавляет сумму к числу в колонке `sum_sale`
4. При событии `UPDATE` триггер:
    1. Вычитает из предыдущего количества товаров (`sales.sales_qty`) новое значение
    2. Полученное число умножает на текущую стоимость (`goods.good_price`)
    3. Прибавляет полученную сумму к текущей сумме в `sales.sales_qty` по данному товару
5. При событии `DELETE` триггер:
    1. Считает сумму по формуле `goods.good_price * sales.sales_qty`, где `sales.sales_qty` взят из последнего значения удаленной записи
    2. Вычитаем полученную сумму из `sales.sales_qty`

# Решение домашнего задания

## ФТ №1: Заполнение таблицы good_sum_mart

```sql
INSERT INTO good_sum_mart (good_name, sum_sale)
SELECT G.good_name, sum(G.good_price * S.sales_qty)
	FROM goods G
	INNER JOIN sales S ON S.good_id = G.goods_id
	GROUP BY G.good_name;
 
-- Проверяем данные
SELECT * FROM good_sum_mart;

/*
        good_name         |   sum_sale   
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
*/
```

