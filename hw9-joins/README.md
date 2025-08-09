# Работа с join'ами, статистикой

Цели:

* знать и уметь применять различные виды join'ов
* строить и анализировать план выполенения запроса
* оптимизировать запрос
* уметь собирать и анализировать статистику для таблицы

# Подготовка к решению домашнего задания

Для домашнего задания предлагаю следующую структуру данных — небольшую базу с информацией о студентах, курсах и их оценках. Для создания
и наполнения БД используйте запросы ниже.

**students** — данные о студентах

```sql
CREATE TABLE students
(
    student_id SERIAL PRIMARY KEY,
    name       VARCHAR(50),
    age        INT
);
```

**courses** — данные о курсах

```sql
CREATE TABLE courses
(
    course_id   SERIAL PRIMARY KEY,
    course_name VARCHAR(100)
);
```

**enrollments** — информация о том, какие студенты на каких курсах учатся, и их оценки

```sql
CREATE TABLE enrollments
(
    enrollment_id SERIAL PRIMARY KEY,
    student_id    INT REFERENCES students (student_id),
    course_id     INT REFERENCES courses (course_id),
    grade         CHAR(1)
);
```

Наполнение таблиц:

```sql
INSERT INTO students (name, age)
VALUES ('Иван Иванов', 20),
       ('Мария Петрова', 21),
       ('Сергей Сидоров', 22),
       ('Дмитрий Замятин', 20);

INSERT INTO courses (course_name)
VALUES ('Математика'),
       ('Физика'),
       ('История');

INSERT INTO enrollments (student_id, course_id, grade)
VALUES (1, 1, 'A'), -- Иван Иванов, Математика
       (1, 2, 'B'), -- Иван Иванов, Физика
       (2, 1, 'C'), -- Мария Петрова, Математика
       (3, 3, 'B'); -- Сергей Сидоров, История
```

# Решение домашнего задания

**Прямое соединение (INNER JOIN)** — студенты и их курсы с оценками

```sql
SELECT s.name student_name,
       c.course_name,
       e.grade
FROM students s
         JOIN enrollments e
              ON e.student_id = s.student_id
         JOIN courses c
              ON c.course_id = e.course_id;
```

или

```sql
SELECT s.name student_name,
       c.course_name,
       e.grade
FROM students s
         JOIN enrollments e USING (student_id)
         JOIN courses c USING (course_id);
```

или

```sql
SELECT s.name student_name,
       c.course_name,
       e.grade
FROM students s
         NATURAL JOIN enrollments e
         NATURAL JOIN courses c;
```

**Левостороннее соединение (LEFT JOIN)** — все студенты с их курсами, включая тех, кто не записан ни на один курс

```sql
SELECT s.name student_name,
       c.course_name,
       e.grade
FROM students s
         NATURAL LEFT JOIN enrollments e
         NATURAL LEFT JOIN courses c;
```

**Правостороннее соединение (RIGHT JOIN)** — все курсы с учащимися, даже если на курс никто не записан

```sql
SELECT c.course_name,
       s.name student_name
FROM students s
         NATURAL RIGHT JOIN enrollments e
         NATURAL RIGHT JOIN courses c
ORDER BY c.course_name;
```

**Кросс-соединение (CROSS JOIN)** — все возможные пары студент-курс

```sql
SELECT s.name student_name,
       c.course_name,
       e.grade
FROM students s
         CROSS JOIN courses c
         CROSS JOIN enrollments e;
```

или

```sql
SELECT s.name student_name,
       c.course_name,
       e.grade
FROM students s,
     courses c,
     enrollments e;
```

**Полное соединение (FULL OUTER JOIN)** — объединить студентов и курсы по записям и показать все, даже если нет соответствий

```sql
SELECT s.name student_name,
       c.course_name,
       e.grade
FROM students s
         NATURAL FULL OUTER JOIN enrollments e
         NATURAL FULL OUTER JOIN courses c;
```

Вывести всех студентов с курсами, если есть, а также показать курсы, на которые не записан никто

```sql
SELECT s.name student_name,
       c.course_name,
       e.grade
FROM students s
         NATURAL LEFT JOIN enrollments e
         NATURAL RIGHT JOIN courses c;
```
