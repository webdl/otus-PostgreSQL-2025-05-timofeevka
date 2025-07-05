# Настройка autovacuum с учетом особенностей производительности
Цели:
* запустить нагрузочный тест pgbench
* настроить параметры autovacuum
* проверить работу autovacuum

# Решение домашнего задания
Исходные данные:
* Виртуальная машина: Ubuntu 24.04.2 LTS, 4 Gb RAM, 2 CPU, 10 Gb SSD
* PostgreSQL: 16.9

ВМ собрана на основе файлов настроек: [mini_db.yml](../deploy/vm/group_vars/mini_db.yml), [inventory.ini](../deploy/vm/inventory.ini)
