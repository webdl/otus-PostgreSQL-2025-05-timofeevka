# Логическая и физическая репликации

Цель: реализовать различные виды репликаций на 4 ВМ.

# Подготовка к решению домашнего задания

## Перенастройка старых ВМ

Ранее мы использовали 2 ВМ в наших экспериментах, а теперь к ним необходимо добавить еще 2 ВМ для организации кластера PG. Перед этим
выровним настройки по текущим ВМ:

Первая ВМ была "жирная", с большим количеством памяти и CPU, так как ранее мы активно проводили замеры производительности. Через систему
управления ВМ приводим ее настройки к параметрам: CPU = 2, RAM = 4. Этих значений хватит для экспериментальной настройки репликации.

Вводим новые параметры ВМ в сервис https://pgtune.leopard.in.ua/ и применяем полученные настройки в файле
[db.yml](../deploy/vm/group_vars/db.yml). Изменения следующие:

```
max_connections = 20
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
work_mem = 37449kB
max_worker_processes = 2
max_parallel_workers = 2
checkpoint_timeout = 60
deadlock_timeout = 1000ms
```

В файле [inventory.ini](../deploy/vm/inventory.ini) переносим хост `pg-slave01` в группу `db`, чтобы управлять им одной конфигурацией.
Файл [mini_db.yml](../deploy/vm/group_vars/mini_db.yml) удаляем, так как он теряет свою актуальность.

Запускаем две ВМ и выполняем команду перенастройки систем:

```shell
cd deploy/vm
ansible-playbook playbooks/main.yml
```

Если всё прошло хорошо — можно приступить к следующему разделу.

## Создание новых ВМ

Путем клонирования ВМ `pg-slave01` создаем `pg-slave02` и `pg-slave03`. Перенастраиваем их имена, IP адреса и т.д. Выполняем команду
ниже и убеждаемся, что сервера в рабочем состоянии с единой конфигурацией:

```shell
ansible-playbook playbooks/main.yml
```

```
PLAY RECAP **********************************************************************************************************
pg-master                  : ok=36   changed=1    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0   
pg-slave01                 : ok=36   changed=1    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0   
pg-slave02                 : ok=36   changed=1    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0   
pg-slave03                 : ok=36   changed=1    unreachable=0    failed=0    skipped=11   rescued=0    ignored=0
```

# Решение домашнего задания
