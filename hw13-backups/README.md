# Резервное копирование и восстановление

Цель: применить логический бэкап. Восстановиться из бэкапа

# Решение домашнего задания

## Подготовка БД для РК

Создадим БД и таблицу, резервную копию которых будет производить и наполним ее данными:

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

## РК с помощью COPY

На сервере БД создадим каталог под бекапы:

```shell
sudo su - postgres
mkdir ~/backups
cd ~/backups
pwd
```

Выполним резервное копирование:

```sql
COPY users TO '/var/lib/postgresql/backups/users.csv' WITH DELIMITER '|' HEADER;
```

Выполним восстановление в другую таблицу:

```sql
CREATE TABLE users_copy (LIKE users INCLUDING ALL);

COPY users_copy FROM '/var/lib/postgresql/backups/users.csv' WITH DELIMITER '|' HEADER;

SELECT * FROM users_copy;
```

## РК с помощью pg_dump без сжатия

Зайдем на сервер и выполним резервное копирование:

```shell
cd ~/backups
pg_dump backups > backups.sql
```

Посмотрим на размер файла:

```shell
stat -c '%s bytes' backups.sql
8708 bytes
```

И на его содержимое:

```shell
cat backups.sql
```

```sql
--
-- PostgreSQL database dump
--

-- Dumped from database version 16.9 (Ubuntu 16.9-0ubuntu0.24.04.1)
-- Dumped by pg_dump version 16.9 (Ubuntu 16.9-0ubuntu0.24.04.1)

SET statement_timeout = 0;
SET lock_timeout = 0;
SET idle_in_transaction_session_timeout = 0;
SET client_encoding = 'UTF8';
SET standard_conforming_strings = on;
SELECT pg_catalog.set_config('search_path', '', false);
SET check_function_bodies = false;
SET xmloption = content;
SET client_min_messages = warning;
SET row_security = off;

SET default_tablespace = '';

SET default_table_access_method = heap;

--
-- Name: users; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.users (
    id integer NOT NULL,
    name character varying(255) NOT NULL,
    email character varying(255) NOT NULL
);


ALTER TABLE public.users OWNER TO postgres;

--
-- Name: users_id_seq; Type: SEQUENCE; Schema: public; Owner: postgres
--

CREATE SEQUENCE public.users_id_seq
    AS integer
    START WITH 1
    INCREMENT BY 1
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;


ALTER SEQUENCE public.users_id_seq OWNER TO postgres;

--
-- Name: users_id_seq; Type: SEQUENCE OWNED BY; Schema: public; Owner: postgres
--

ALTER SEQUENCE public.users_id_seq OWNED BY public.users.id;


--
-- Name: users_copy; Type: TABLE; Schema: public; Owner: postgres
--

CREATE TABLE public.users_copy (
    id integer DEFAULT nextval('public.users_id_seq'::regclass) NOT NULL,
    name character varying(255) NOT NULL,
    email character varying(255) NOT NULL
);


ALTER TABLE public.users_copy OWNER TO postgres;

--
-- Name: users id; Type: DEFAULT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.users ALTER COLUMN id SET DEFAULT nextval('public.users_id_seq'::regclass);


--
-- Data for Name: users; Type: TABLE DATA; Schema: public; Owner: postgres
--

COPY public.users (id, name, email) FROM stdin;
1	User1	User1@example.com
2	User2	User2@example.com
3	User3	User3@example.com
4	User4	User4@example.com
5	User5	User5@example.com
6	User6	User6@example.com
7	User7	User7@example.com
8	User8	User8@example.com
9	User9	User9@example.com
10	User10	User10@example.com
11	User11	User11@example.com
12	User12	User12@example.com
13	User13	User13@example.com
14	User14	User14@example.com
15	User15	User15@example.com
16	User16	User16@example.com
17	User17	User17@example.com
18	User18	User18@example.com
19	User19	User19@example.com
20	User20	User20@example.com
21	User21	User21@example.com
22	User22	User22@example.com
23	User23	User23@example.com
24	User24	User24@example.com
25	User25	User25@example.com
26	User26	User26@example.com
27	User27	User27@example.com
28	User28	User28@example.com
29	User29	User29@example.com
30	User30	User30@example.com
31	User31	User31@example.com
32	User32	User32@example.com
33	User33	User33@example.com
34	User34	User34@example.com
35	User35	User35@example.com
36	User36	User36@example.com
37	User37	User37@example.com
38	User38	User38@example.com
39	User39	User39@example.com
40	User40	User40@example.com
41	User41	User41@example.com
42	User42	User42@example.com
43	User43	User43@example.com
44	User44	User44@example.com
45	User45	User45@example.com
46	User46	User46@example.com
47	User47	User47@example.com
48	User48	User48@example.com
49	User49	User49@example.com
50	User50	User50@example.com
51	User51	User51@example.com
52	User52	User52@example.com
53	User53	User53@example.com
54	User54	User54@example.com
55	User55	User55@example.com
56	User56	User56@example.com
57	User57	User57@example.com
58	User58	User58@example.com
59	User59	User59@example.com
60	User60	User60@example.com
61	User61	User61@example.com
62	User62	User62@example.com
63	User63	User63@example.com
64	User64	User64@example.com
65	User65	User65@example.com
66	User66	User66@example.com
67	User67	User67@example.com
68	User68	User68@example.com
69	User69	User69@example.com
70	User70	User70@example.com
71	User71	User71@example.com
72	User72	User72@example.com
73	User73	User73@example.com
74	User74	User74@example.com
75	User75	User75@example.com
76	User76	User76@example.com
77	User77	User77@example.com
78	User78	User78@example.com
79	User79	User79@example.com
80	User80	User80@example.com
81	User81	User81@example.com
82	User82	User82@example.com
83	User83	User83@example.com
84	User84	User84@example.com
85	User85	User85@example.com
86	User86	User86@example.com
87	User87	User87@example.com
88	User88	User88@example.com
89	User89	User89@example.com
90	User90	User90@example.com
91	User91	User91@example.com
92	User92	User92@example.com
93	User93	User93@example.com
94	User94	User94@example.com
95	User95	User95@example.com
96	User96	User96@example.com
97	User97	User97@example.com
98	User98	User98@example.com
99	User99	User99@example.com
100	User100	User100@example.com
\.


--
-- Data for Name: users_copy; Type: TABLE DATA; Schema: public; Owner: postgres
--

COPY public.users_copy (id, name, email) FROM stdin;
1	User1	User1@example.com
2	User2	User2@example.com
3	User3	User3@example.com
4	User4	User4@example.com
5	User5	User5@example.com
6	User6	User6@example.com
7	User7	User7@example.com
8	User8	User8@example.com
9	User9	User9@example.com
10	User10	User10@example.com
11	User11	User11@example.com
12	User12	User12@example.com
13	User13	User13@example.com
14	User14	User14@example.com
15	User15	User15@example.com
16	User16	User16@example.com
17	User17	User17@example.com
18	User18	User18@example.com
19	User19	User19@example.com
20	User20	User20@example.com
21	User21	User21@example.com
22	User22	User22@example.com
23	User23	User23@example.com
24	User24	User24@example.com
25	User25	User25@example.com
26	User26	User26@example.com
27	User27	User27@example.com
28	User28	User28@example.com
29	User29	User29@example.com
30	User30	User30@example.com
31	User31	User31@example.com
32	User32	User32@example.com
33	User33	User33@example.com
34	User34	User34@example.com
35	User35	User35@example.com
36	User36	User36@example.com
37	User37	User37@example.com
38	User38	User38@example.com
39	User39	User39@example.com
40	User40	User40@example.com
41	User41	User41@example.com
42	User42	User42@example.com
43	User43	User43@example.com
44	User44	User44@example.com
45	User45	User45@example.com
46	User46	User46@example.com
47	User47	User47@example.com
48	User48	User48@example.com
49	User49	User49@example.com
50	User50	User50@example.com
51	User51	User51@example.com
52	User52	User52@example.com
53	User53	User53@example.com
54	User54	User54@example.com
55	User55	User55@example.com
56	User56	User56@example.com
57	User57	User57@example.com
58	User58	User58@example.com
59	User59	User59@example.com
60	User60	User60@example.com
61	User61	User61@example.com
62	User62	User62@example.com
63	User63	User63@example.com
64	User64	User64@example.com
65	User65	User65@example.com
66	User66	User66@example.com
67	User67	User67@example.com
68	User68	User68@example.com
69	User69	User69@example.com
70	User70	User70@example.com
71	User71	User71@example.com
72	User72	User72@example.com
73	User73	User73@example.com
74	User74	User74@example.com
75	User75	User75@example.com
76	User76	User76@example.com
77	User77	User77@example.com
78	User78	User78@example.com
79	User79	User79@example.com
80	User80	User80@example.com
81	User81	User81@example.com
82	User82	User82@example.com
83	User83	User83@example.com
84	User84	User84@example.com
85	User85	User85@example.com
86	User86	User86@example.com
87	User87	User87@example.com
88	User88	User88@example.com
89	User89	User89@example.com
90	User90	User90@example.com
91	User91	User91@example.com
92	User92	User92@example.com
93	User93	User93@example.com
94	User94	User94@example.com
95	User95	User95@example.com
96	User96	User96@example.com
97	User97	User97@example.com
98	User98	User98@example.com
99	User99	User99@example.com
100	User100	User100@example.com
\.


--
-- Name: users_id_seq; Type: SEQUENCE SET; Schema: public; Owner: postgres
--

SELECT pg_catalog.setval('public.users_id_seq', 101, true);


--
-- Name: users_copy users_copy_email_key; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.users_copy
    ADD CONSTRAINT users_copy_email_key UNIQUE (email);


--
-- Name: users_copy users_copy_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.users_copy
    ADD CONSTRAINT users_copy_pkey PRIMARY KEY (id);


--
-- Name: users users_email_key; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.users
    ADD CONSTRAINT users_email_key UNIQUE (email);


--
-- Name: users users_pkey; Type: CONSTRAINT; Schema: public; Owner: postgres
--

ALTER TABLE ONLY public.users
    ADD CONSTRAINT users_pkey PRIMARY KEY (id);


--
-- PostgreSQL database dump complete
--
```

## РК с помощью pg_dump со сжатием

Зайдем на сервер и выполним резервное копирование:

```shell
cd ~/backups
pg_dump -Fc -Z 9 backups > backups.dump

stat -c '%s bytes' backups.dump
9139 bytes
```

И второй вариант, когда для сжатия используется внешняя утилита gzip:

```shell
cd ~/backups
pg_dump -Fc backups | gzip > backups.dump.gz

stat -c '%s bytes' backups.dump.gz
1676 bytes
```

Видим, что использование ключа `-Z 9` (максимальное сжатие) в `pg_dump` не дает такого же эффекта, как использование gzip со сжатием 
по-умолчанию (6). Это связано с тем, что встроенное сжатие pg_dump «на лету» происходит во время записи дампа, что может ограничивать 
эффективность из-за особенностей потока и взаимодействия с форматом дампа. В то же время внешний gzip работает с полным файлом и 
может более эффективно оптимизировать сжатие, так как оперирует целым файлом, а не блоками данных.
