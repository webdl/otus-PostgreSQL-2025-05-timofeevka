# Установка PostgreSQL на виртуальный сервер
Для выполнения домашнего задания на виртуальной машине (ВМ) был установлен сервер PostgreSQL 16. Этот процесс описан в 
[deploy/vm/](../deploy/vm). Сервер PostgreSQL уже сконфигурирован на внешнее подключение, поэтому достаточно настроить соединение с ним 
из DBeaver или PgAdmin4 по логину и паролю `postgres`. Пример настройки PgAdmin4 смотрите в [ДЗ №2](../hw2-install-on-docker).

# Выполнение домашнего задачи
## Создание данных в БД
Создать новую БД:
```sql
CREATE DATABASE persons;
```
Подключиться к ней и создать произвольную таблицу:
```sql
START TRANSACTION;

CREATE TABLE test(
	id SERIAL NOT NULL PRIMARY KEY,
	text TEXT
);

INSERT INTO test (text) VALUES('Hello, World!');
INSERT INTO test (text) VALUES('Foo, bar!');

COMMIT;
```

## Подключение нового диска к ВМ
Остановить ВМ. В среде виртуализации создать новый диск, подключить к ВМ и запустить ее.

Проверить, подключен ли диск в ОС:
```shell
> lsblk -f
NAME                      FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
vda
├─vda1                    vfat        FAT32          4E1D-6B6F                               944.8M     1% /boot/efi
├─vda2                    ext4        1.0            da24c1c5-9d6a-46a8-af3a-2dcb68e2c044      1.4G    11% /boot
└─vda3                    LVM2_member LVM2 001       Kl3nbn-ITet-IhKv-3IMB-hbED-sRds-M3DrzZ
  └─ubuntu--vg-ubuntu--lv ext4        1.0            11e4b9fe-b933-4e1f-a799-db74e6d4e660        6G    33% /
vdb
```
Диск найден под именем `vdb`.

Создать раздел:
```shell
> sudo fdisk /dev/vdb

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-10485759, default 2048): 2048
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-10485759, default 10485759): 10485759

Created a new partition 1 of type 'Linux' and of size 5 GiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```

Проверить еще раз:
```shell
> lsblk -f
NAME                      FSTYPE      FSVER    LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
vda
├─vda1                    vfat        FAT32          4E1D-6B6F                               944.8M     1% /boot/efi
├─vda2                    ext4        1.0            da24c1c5-9d6a-46a8-af3a-2dcb68e2c044      1.4G    11% /boot
└─vda3                    LVM2_member LVM2 001       Kl3nbn-ITet-IhKv-3IMB-hbED-sRds-M3DrzZ
  └─ubuntu--vg-ubuntu--lv ext4        1.0            11e4b9fe-b933-4e1f-a799-db74e6d4e660        6G    33% /
vdb
└─vdb1
```

Отформатировать:
```shell
> sudo mkfs.ext4 /dev/vdb1
mke2fs 1.47.0 (5-Feb-2023)
Discarding device blocks: done
Creating filesystem with 1310464 4k blocks and 327680 inodes
Filesystem UUID: d0800cb8-28f6-4b21-9b4b-ca8ba908edc2
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
```

Смонтировать:
```shell
> sudo mkdir /mnt/pg_data
> sudo mount /dev/vdb1 /mnt/pg_data
```

Настроить fstab:
```shell
> sudo blkid /dev/vdb1
/dev/vdb1: UUID="d0800cb8-28f6-4b21-9b4b-ca8ba908edc2" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="92d9a293-01"

> echo "UUID=d0800cb8-28f6-4b21-9b4b-ca8ba908edc2 /mnt/mydisk ext4 defaults 0 2" | sudo tee -a /etc/fstab
> sudo reboot
```

Дождаться перезагрузки и проверить диск:
```shell
> df -lh
Filesystem                         Size  Used Avail Use% Mounted on
tmpfs                              591M  1.3M  590M   1% /run
efivarfs                           256K   27K  230K  11% /sys/firmware/efi/efivars
/dev/mapper/ubuntu--vg-ubuntu--lv  9.8G  3.3G  6.0G  36% /
tmpfs                              2.9G  1.1M  2.9G   1% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
/dev/vdb1                          4.9G   24K  4.6G   1% /mnt/pg_data     # <---- новый диск
/dev/vda2                          1.7G  191M  1.4G  12% /boot
/dev/vda1                          952M  6.4M  945M   1% /boot/efi
tmpfs                              591M   12K  591M   1% /run/user/1000
```
