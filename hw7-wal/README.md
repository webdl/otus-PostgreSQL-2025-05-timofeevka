# Работа с журналами
Цели:
* уметь работать с журналами и контрольными точками
* уметь настраивать параметры журналов

# Решение домашнего задания
## Подготовка сервера
Перед началом работ следует очистить каталог с текущими wal-файлами, если используется не свежеустановленный сервер.

### Немного теории
WAL-файлы в PostgreSQL удаляются после того, как они становятся ненужными для восстановления и репликации. Основные моменты:
- WAL-файлы хранятся в каталоге `pg_wal` и имеют фиксированный размер (обычно 16МБ, согласно параметру `wal_buffers`).
- При запуске **контрольной точки (checkpoint)** PostgreSQL фиксирует состояние базы, обновляет файлы данных и оценивает, сколько 
  сегментов WAL нужно сохранить для восстановления.
- После контрольной точки старые WAL-файлы, которые уже не нужны для восстановления, **перерабатываются (переименовываются и повторно 
  используются) или удаляются**, если их количество превышает минимально необходимое `min_wal_size` и не превышает максимально 
  допустимое `max_wal_size`
- Если размер WAL на диске превышает `min_wal_size`, старые сегменты WAL удаляются при освобождении пространства, иначе они 
  перезаписываются.
- В случае репликации WAL-файлы не удаляются, пока все реплики не подтвердят получение и применение соответствующих записей. Если 
  репликация отстает, файлы накапливаются, чтобы обеспечить возможность восстановления на репликах
- Для очистки архивированных WAL-файлов можно использовать утилиту `pg_archivecleanup`, которая удаляет файлы, архивированные до 
  определённой точки резервного копирования

Но, это всё в теории... На практике же оказывается, что PostgreSQL очень неохотно расстается с захваченным им местом. Об этом далее.

### Сокращение количества wal-файлов с помощью функции
Для начала перенастройкой параметров `min_wal_size` и `max_wal_size`, которые в моем случае равны 1Gb и 4Gb соответственно. Так как сервер 
активно работал до этого, то в каталоге `/mnt/pg_data/main/pg_wal` скопилось 3.7Gb wal-файлов.

В файле `postgresql.conf` задаем параметры:
```
max_wal_size = '200MB'
min_wal_size = '100MB'
```
и перезапускаем сервер:
```shell
> pg_ctlcluster 16 main restart
```

К сожалению этого недостаточно, чтобы PostgreSQL освободил занимаемое место. И даже выполнение команды `CHECKPOINT;` тоже не поможет. 

Причина такого поведения описана в пункте 3 раздела Немного теории выше — пока PostgreSQL не пройдется по этим файлам повторно, то есть 
не задействует эти wal-файлы для записи новых журналов — он не удалит их. Если у вас сервер с малым трафиком, например, сервер для 
разработки, и вы недавно выполнили операцию, которая создала много записей (например, импорт данных), то количество wal-файлов может 
вырасти вплоть до максимально разрешённого размера (max_wal_size).

Чтобы принудительно заставить сервер пройтись по всем существующим wal-файлам и удалить лишние, можно использовать специальную функцию, 
которая повторно вызывает команду переключения на новый wal-файл (pg_switch_wal()). Также запускается контрольная точка (checkpoint),
которая записывает текущие изменения на диск и «закрывает» старый wal-файл. Делая это, сервер уменьшит количество wal-файлов до текущего 
значения max_wal_size или даже меньше.

Однако подобное действие может быть нежелательным на работающем сервере, особенно если у вас настроено архивирование WAL — оно может 
замедлиться или работать некорректно. Поэтому эту процедуру стоит применять только если ваш сервер соответствует описанной ситуации 
(малый трафик, много старых WAL-файлов после масштабной операции). В зависимости от того, сколько у вас файлов WAL, выполнение этой 
операции может занять некоторое время.

```sql
create or replace function pg_wal_cycle_all() 
returns int language plpgsql 
as $$
declare
    wal_count int;
    wal_seg varchar;
begin 
    select count(*) - 1 
    into wal_count 
    from pg_ls_dir('pg_wal');

    for wal in 1..wal_count loop 
        select pg_walfile_name(pg_switch_wal()) into wal_seg;
        raise notice 'segment %', wal_seg;
        checkpoint;
    end loop;
    return wal_count;
end;$$;

-- Выполнить функцию
select pg_wal_cycle_all();
```

### Сокращение количества wal-файлов с pgbench
Как альтернативный вариант вышеописанному способу. Если запустить его на любой БД (так как wal-файлы общие для всего сервера), то он 
своей работой заполнит wal-файлы и PostgreSQL удалит пройденные файлы. Пример:
```shell
> pgbench -i -s 10 persons
> pgbench -c 10 -j 2 -t 50000 persons
```

## Настройка сервера
Вернем предыдущие настройки `min_wal_size` и `max_wal_size` и зададим параметр `checkpoint_timeout` равный 30. Также, чтобы отслеживать 
выполнение контрольных точек, выставим параметры `logging_collector` и `log_checkpoints` в значение "on" (по умолчанию последний уже задан,
но не будет лишним прописать его принудительно). Все настройки производятся через файл [db.yml](../deploy/vm/group_vars/db.yml).

Срез измененных настроек:
```
min_wal_size = 1GB
max_wal_size = 4GB
logging_collector = on
log_checkpoints = on
```

Применим настройки через команду ниже и дождемся перезапуска кластера:
```shell
> ansible-playbook playbooks/install_db.yml -l db
```

### Проверка логирования контрольных точек
Найдем расположение лог-файлов:
```sql
SHOW data_directory; -- каталог с данными
SHOW log_directory;  -- каталог с лог-файлами в каталоге с данными
```
В моем случае лог-файлы находятся по пути `/mnt/pg_data/main/log`, а сам лог-файл называется `postgresql-2025-08-02_093617.log`.

Выполним shell-команду потокового чтения файла:
```shell
> tail -f /mnt/pg_data/main/log/postgresql-2025-08-02_093617.log
```

Выполним принудительную запись контрольной точки:
```sql
CHECKPOINT;
```

Увидим, что контрольные точки логируются:
```
2025-08-02 09:40:49.732 UTC [7221] LOG:  checkpoint starting: immediate force wait
2025-08-02 09:40:49.747 UTC [7221] LOG:  checkpoint complete: wrote 0 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.007 s, sync=0.001 s, total=0.016 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=0 kB; lsn=32/CD36E7A8, redo lsn=32/CD36E770
```

## Измерение контрольных точек при работе pgbench
Наша задача измерить, какой объект wal-файлов будет сгенерирован при работе pgbench в течение 10 минут, а также вычислить сколько в 
среднем создается wal-файлов между контрольными точками.

Для начала посмотрим текущее количество wal-файлов:
```shell
> find /mnt/pg_data/main/pg_wal -maxdepth 1 -type f | wc -l
12
```

Выполним shell-команду потокового чтения лог-файла:
```shell
> tail -f /mnt/pg_data/main/log/postgresql-2025-08-02_093617.log
```

В другом окне терминала запускаем pgbench на 10 минут:
```shell
> pgbench -c 10 -j 10 -T 600 persons
```
![](img/spongebob.jpeg)

### Проверяем результаты
Количество wal-файлов увеличилось до 30 штук:
```shell
> find /mnt/pg_data/main/pg_wal -maxdepth 1 -type f | wc -l
30
```

В лог-файле появилась следующая информация:
```
2025-08-02 10:44:23.827 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:44:50.029 UTC [7221] LOG:  checkpoint complete: wrote 18662 buffers (3.6%); 0 WAL file(s) added, 0 removed, 10 recycled; write=26.172 s, sync=0.006 s, total=26.202 s; sync files=15, longest=0.002 s, average=0.001 s; distance=167692 kB, estimate=167692 kB; lsn=32/E4D9EA78, redo lsn=32/D7733640
2025-08-02 10:44:53.031 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:45:20.030 UTC [7221] LOG:  checkpoint complete: wrote 19694 buffers (3.8%); 0 WAL file(s) added, 0 removed, 14 recycled; write=26.965 s, sync=0.003 s, total=26.999 s; sync files=7, longest=0.002 s, average=0.001 s; distance=229647 kB, estimate=229647 kB; lsn=32/F3569938, redo lsn=32/E5777568
2025-08-02 10:45:23.033 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:45:50.043 UTC [7221] LOG:  checkpoint complete: wrote 20340 buffers (3.9%); 0 WAL file(s) added, 0 removed, 14 recycled; write=26.971 s, sync=0.008 s, total=27.011 s; sync files=17, longest=0.004 s, average=0.001 s; distance=237439 kB, estimate=237439 kB; lsn=33/19542E8, redo lsn=32/F3F571C8
2025-08-02 10:45:53.046 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:46:20.045 UTC [7221] LOG:  checkpoint complete: wrote 19627 buffers (3.7%); 0 WAL file(s) added, 0 removed, 15 recycled; write=26.964 s, sync=0.006 s, total=26.999 s; sync files=7, longest=0.004 s, average=0.001 s; distance=233376 kB, estimate=237032 kB; lsn=33/1025C620, redo lsn=33/233F2F8
2025-08-02 10:46:23.048 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:46:50.062 UTC [7221] LOG:  checkpoint complete: wrote 20342 buffers (3.9%); 0 WAL file(s) added, 0 removed, 14 recycled; write=26.975 s, sync=0.008 s, total=27.014 s; sync files=18, longest=0.004 s, average=0.001 s; distance=238538 kB, estimate=238538 kB; lsn=33/1E5CEFB8, redo lsn=33/10C31DA8
2025-08-02 10:46:53.065 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:47:20.073 UTC [7221] LOG:  checkpoint complete: wrote 19403 buffers (3.7%); 0 WAL file(s) added, 0 removed, 14 recycled; write=26.969 s, sync=0.007 s, total=27.009 s; sync files=7, longest=0.004 s, average=0.001 s; distance=233136 kB, estimate=237998 kB; lsn=33/2CFC8D08, redo lsn=33/1EFDDE48
2025-08-02 10:47:23.077 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:47:50.090 UTC [7221] LOG:  checkpoint complete: wrote 20262 buffers (3.9%); 0 WAL file(s) added, 0 removed, 15 recycled; write=26.974 s, sync=0.008 s, total=27.014 s; sync files=17, longest=0.004 s, average=0.001 s; distance=239397 kB, estimate=239397 kB; lsn=33/3B47B198, redo lsn=33/2D9A7590
2025-08-02 10:47:53.093 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:48:20.091 UTC [7221] LOG:  checkpoint complete: wrote 19292 buffers (3.7%); 0 WAL file(s) added, 0 removed, 14 recycled; write=26.963 s, sync=0.006 s, total=26.999 s; sync files=7, longest=0.004 s, average=0.001 s; distance=234189 kB, estimate=238876 kB; lsn=33/49D39108, redo lsn=33/3BE5AB80
2025-08-02 10:48:23.093 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:48:50.101 UTC [7221] LOG:  checkpoint complete: wrote 20101 buffers (3.8%); 0 WAL file(s) added, 0 removed, 15 recycled; write=26.969 s, sync=0.008 s, total=27.008 s; sync files=18, longest=0.004 s, average=0.001 s; distance=238358 kB, estimate=238825 kB; lsn=33/582E56B8, redo lsn=33/4A720440
2025-08-02 10:48:53.104 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:49:20.101 UTC [7221] LOG:  checkpoint complete: wrote 19190 buffers (3.7%); 0 WAL file(s) added, 0 removed, 14 recycled; write=26.964 s, sync=0.005 s, total=26.998 s; sync files=7, longest=0.004 s, average=0.001 s; distance=235213 kB, estimate=238463 kB; lsn=33/66B9CEC0, redo lsn=33/58CD3A48
2025-08-02 10:49:23.105 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:49:50.119 UTC [7221] LOG:  checkpoint complete: wrote 20046 buffers (3.8%); 0 WAL file(s) added, 0 removed, 15 recycled; write=26.971 s, sync=0.008 s, total=27.015 s; sync files=17, longest=0.004 s, average=0.001 s; distance=238286 kB, estimate=238446 kB; lsn=33/750ED888, redo lsn=33/67587400
2025-08-02 10:49:53.122 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:50:20.124 UTC [7221] LOG:  checkpoint complete: wrote 19150 buffers (3.7%); 0 WAL file(s) added, 0 removed, 14 recycled; write=26.962 s, sync=0.006 s, total=27.002 s; sync files=7, longest=0.004 s, average=0.001 s; distance=234803 kB, estimate=238081 kB; lsn=33/839B97F8, redo lsn=33/75AD4100
2025-08-02 10:50:23.126 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:50:50.041 UTC [7221] LOG:  checkpoint complete: wrote 20010 buffers (3.8%); 0 WAL file(s) added, 0 removed, 15 recycled; write=26.872 s, sync=0.009 s, total=26.915 s; sync files=18, longest=0.004 s, average=0.001 s; distance=238451 kB, estimate=238451 kB; lsn=33/91F91CB0, redo lsn=33/843B1000
2025-08-02 10:50:53.044 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:51:20.043 UTC [7221] LOG:  checkpoint complete: wrote 19068 buffers (3.6%); 0 WAL file(s) added, 0 removed, 14 recycled; write=26.960 s, sync=0.007 s, total=27.000 s; sync files=7, longest=0.004 s, average=0.001 s; distance=235388 kB, estimate=238145 kB; lsn=33/A0905708, redo lsn=33/92990140
2025-08-02 10:51:23.045 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:51:50.068 UTC [7221] LOG:  checkpoint complete: wrote 20007 buffers (3.8%); 0 WAL file(s) added, 0 removed, 15 recycled; write=26.976 s, sync=0.007 s, total=27.023 s; sync files=16, longest=0.003 s, average=0.001 s; distance=239054 kB, estimate=239054 kB; lsn=33/AEE94708, redo lsn=33/A1303B98
2025-08-02 10:51:53.071 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:52:20.067 UTC [7221] LOG:  checkpoint complete: wrote 19012 buffers (3.6%); 0 WAL file(s) added, 0 removed, 14 recycled; write=26.955 s, sync=0.005 s, total=26.997 s; sync files=6, longest=0.003 s, average=0.001 s; distance=235019 kB, estimate=238651 kB; lsn=33/BD9596E0, redo lsn=33/AF886A10
2025-08-02 10:52:23.070 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:52:50.099 UTC [7221] LOG:  checkpoint complete: wrote 23084 buffers (4.4%); 0 WAL file(s) added, 0 removed, 15 recycled; write=26.985 s, sync=0.008 s, total=27.029 s; sync files=17, longest=0.003 s, average=0.001 s; distance=240453 kB, estimate=240453 kB; lsn=33/CBF172F8, redo lsn=33/BE357ED8
2025-08-02 10:52:53.103 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:53:20.095 UTC [7221] LOG:  checkpoint complete: wrote 19080 buffers (3.6%); 0 WAL file(s) added, 0 removed, 14 recycled; write=26.949 s, sync=0.007 s, total=26.993 s; sync files=7, longest=0.004 s, average=0.001 s; distance=235123 kB, estimate=239920 kB; lsn=33/DA8A1978, redo lsn=33/CC8F4D78
2025-08-02 10:53:23.098 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:53:50.109 UTC [7221] LOG:  checkpoint complete: wrote 19954 buffers (3.8%); 0 WAL file(s) added, 0 removed, 15 recycled; write=26.971 s, sync=0.006 s, total=27.012 s; sync files=16, longest=0.003 s, average=0.001 s; distance=239301 kB, estimate=239858 kB; lsn=33/E8F25890, redo lsn=33/DB2A6270
2025-08-02 10:53:53.112 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:54:20.124 UTC [7221] LOG:  checkpoint complete: wrote 18997 buffers (3.6%); 0 WAL file(s) added, 0 removed, 14 recycled; write=26.964 s, sync=0.018 s, total=27.013 s; sync files=6, longest=0.010 s, average=0.003 s; distance=236188 kB, estimate=239491 kB; lsn=33/F62410E0, redo lsn=33/E994D3A8
2025-08-02 10:55:23.195 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 10:55:50.060 UTC [7221] LOG:  checkpoint complete: wrote 18184 buffers (3.5%); 0 WAL file(s) added, 0 removed, 13 recycled; write=26.829 s, sync=0.012 s, total=26.866 s; sync files=17, longest=0.007 s, average=0.001 s; distance=205794 kB, estimate=236121 kB; lsn=33/F6245E60, redo lsn=33/F6245E28
2025-08-02 11:31:24.921 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 11:31:25.048 UTC [7221] LOG:  checkpoint complete: wrote 1 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.108 s, sync=0.006 s, total=0.128 s; sync files=1, longest=0.006 s, average=0.006 s; distance=6 kB, estimate=212510 kB; lsn=33/F62479D0, redo lsn=33/F6247998
2025-08-02 11:31:54.080 UTC [7221] LOG:  checkpoint starting: time
2025-08-02 11:31:54.306 UTC [7221] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.211 s, sync=0.005 s, total=0.226 s; sync files=1, longest=0.005 s, average=0.005 s; distance=13 kB, estimate=191260 kB; lsn=33/F624B140, redo lsn=33/F624B108
```
**Где:**
* **wrote 18997 buffers (3.6%)** — кичество страниц из буферного кэша (размером 8 KB), которые были записаны на диск в рамках 
контрольной точки. В скобках — процент от общего кэша буферов, который был перезаписан.
* **0 WAL file(s) added, 0 removed, 13 recycled** — операции с WAL-файлами
  * 0 новых wal-файлов было создано в момент контрольной точки. Создание новых сегментов происходит, если в процессе работы 
    системы объём WAL превысил текущий параметр `wal_buffers`.
  * 0 wal-файлов было удалено. После успешной контрольной точки PostgreSQL может убрать старые wal-файлы, которые уже не нужны для 
    восстановления или репликации. В данном случае 0 потому что `max_wal_size = 4GB`
  * 13 wal-файлов переиспользовано. При переиспользовании старый wal-файл перезаписывается заново, уменьшая нагрузку на файловую систему и 
    снижая количество созданы/удалённых файлов.
* **write=26.829 s** — время, затраченное на фактическую запись данных (страниц) на диск во время контрольной точки, в секундах.
* **sync=0.012 s** — время затраченное на синхронизацию записанных данных с диском (fsync), чтобы гарантировать их физическую записанность.
* **total=26.866 s** — общее время контрольной точки — сумма всех операций, включая проверку и управление процессом.
* **sync files=17** — количество файлов, для которых вызвана синхронизация (fsync) в ходе контрольной точки.
* **longest=0.007 s, average=0.001 s** — время самой длительной и средней операции fsync для файлов в контрольной точке.
* **distance=205794 kB** — объём данных в килобайтах, пройденных с момента предыдущей контрольной точки (расстояние по WAL между 
контрольными точками). Это мера объёма изменений.
* **estimate=236121 kB** — оценочный объём данных, которые система планирует записать / пройти к следующей контрольной точке (примерная 
оценка).
* **lsn=33/F6245E60, redo lsn=33/F6245E28**:
  * lsn — Log Sequence Number (положение контрольной точки в журнале WAL). Текущий LSN после выполнения контрольной точки.
  * redo lsn — минимальный LSN, начиная с которого должен выполняться восстановительный процесс при запуске после сбоя, чтобы включить 
   все изменения после контрольной точки.

## Итоги
Отвечая на поставленный вопросы в начале темы, можно сказать, что для работы pgbech с параметрами выше понадобилось 30 wal-файлов, а 
среднее количество используемых wal-файлов между контрольными точками ~15.

При этом можно заметить, что время выполнения контрольных точек было ~27 секунд, что очень близко к заданному ранее значению в 30 секунд.
Это говорит нам о том, что при увеличении нагрузки на кластер PostgreSQL или уменьшении параметра `checkpoint_timeout` наши контрольные 
точки начнут выполняться с задержкой.
