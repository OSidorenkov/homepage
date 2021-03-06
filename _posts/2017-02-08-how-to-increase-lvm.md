---
layout: post
title:  "Увеличить дисковое пространство виртуальной машины Linux в hyper-v"
date:   2017-02-08
desc: "Как увеличить LVM раздел OS Linux CentOS"
keywords: "CentOS,LVM,увеличить диск,Linux, Hyper-V"
categories: [Linux]
tags: [CentOS,LVM,Hyper-V]
icon: icon-centos
---

В жизни случаются разные ситуации, в следствии которых приходится увеличивать размер жесткого диска системы. Сразу проясню: имеется CentOS 7.2 (используются LVM тома) на Hyper-V. Увеличить размер диска виртуальной машины можно только при условии, что изначально он был создан динамически расширяемым, а увеличить размер файловой системы гостевой ОС, только если ОС установлена на LVM разделы.

Заходим в Диспетчер Hyper-V и выбираем нужную виртуальную машину, в параметрах жесткого диска выбираем "Правка"  
	<img src="{{ site.img_path }}/lvm/image2016-8-29_8-54-46.jpg" width="30%" class="image">

Имейте в виду, что если у вас есть контрольные точки для этой машины, то расширить диск не получится.  
Итак, в действиях выбираем "Развернуть" и устанавливаем нужный размер:  
	<img src="{{ site.img_path }}/lvm/image2016-8-29_8-59-59.jpg" width="30%" class="image">

Пространство мы выделили, теперь нужно его добавить к имеющемуся разделу, т.е задача смонтировать в "/". Смотрим что мы имеем:

```
root /home/OSSidorenkov # df -h
Файловая система        Размер Использовано  Дост Использовано% Cмонтировано в
/dev/mapper/centos-root   123G         2,7G  114G            3% /
devtmpfs                  896M            0  896M            0% /dev
tmpfs                     906M            0  906M            0% /dev/shm
tmpfs                     906M         8,3M  898M            1% /run
tmpfs                     906M            0  906M            0% /sys/fs/cgroup
/dev/sda2                 494M         159M  336M           33% /boot
/dev/sda1                 200M         9,5M  191M            5% /boot/efi
tmpfs                     182M            0  182M            0% /run/user/0

root /home/OSSidorenkov # fdisk -l
Disk /dev/sda: 161.1 GB, 161061273600 bytes, 314572800 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disk label type: dos
Disk identifier: 0x00000000
Устр-во Загр     Начало       Конец       Блоки   Id  Система
/dev/sda1               1   266338303   133169151+  ee  GPT
Partition 1 does not start on physical sector boundary.
Disk /dev/mapper/centos-root: 133.5 GB, 133475336192 bytes, 260694016 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disk /dev/mapper/centos-swap: 2147 MB, 2147483648 bytes, 4194304 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
```

Кстати, сразу отсюда берем имя группы томов centos и имя тома root, и запоменаем эти имена. У вас они будут другие.

Т.к. у нас теперь имеется неразмеченная область, то создадим новый раздел sda3 с типом раздела Linux LVM (код типа 8e) на этой области. Для этого начинаем работу с устройством sda. Для создании партиции, воспользуемся утилитой gdisk:

Для создании партиции, воспользуемся утилитой gdisk. Вообще можно использовать также и fdisk, но почему то после ребута машины, разбитое пространство не сохраняется. Возможно дело в самой системе виртуализации hyper-v. 

`yum install gdisk -y`

Создаем раздел: 

```
#root /home/OSSidorenkov # gdisk /dev/sda
GPT fdisk (gdisk) version 0.8.6
Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present
Found valid GPT with protective MBR; using GPT.
Command (? for help): n
Partition number (4-128, default 4):
First sector (34-266338270, default = 266336256) or {+-}size{KMGTP}:
Last sector (266336256-266338270, default = 266338270) or {+-}size{KMGTP}:
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 8e00
Changed type of partition to 'Linux LVM'
Command (? for help): p
Disk /dev/sda: 314572800 sectors, 150.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): D2DF9F8F-951C-4EF7-A074-1D7F42280F74
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 266338270
Partitions will be aligned on 2048-sector boundaries
Total free space is 2014 sectors (1007.0 KiB)
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048          411647   200.0 MiB   EF00  EFI System Partition
   2          411648         1435647   500.0 MiB   0700
   3         1435648       266336255   126.3 GiB   8E00
   4       266336256       266338270   1007.5 KiB  8E00  Linux LVM
Command (? for help): w
Warning! Secondary header is placed too early on the disk! Do you want to
correct this problem? (Y/N): y
Have moved second header and partition table to correct location.
Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!
Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/sda.
Warning: The kernel is still using the old partition table.
The new table will be used at the next reboot.
The operation has completed successfully.
```
Здесь видно, что партиция создалась, но не то что хотелось.. так что проделываем ту же операцию еще раз: 

```
root /home/OSSidorenkov # gdisk /dev/sda
GPT fdisk (gdisk) version 0.8.6
Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present
Found valid GPT with protective MBR; using GPT.
Command (? for help): n
Partition number (5-128, default 5):
First sector (34-314572766, default = 266338304) or {+-}size{KMGTP}:
Last sector (266338304-314572766, default = 314572766) or {+-}size{KMGTP}:
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 8e00
Changed type of partition to 'Linux LVM'
Command (? for help): p
Disk /dev/sda: 314572800 sectors, 150.0 GiB
Logical sector size: 512 bytes
Disk identifier (GUID): D2DF9F8F-951C-4EF7-A074-1D7F42280F74
Partition table holds up to 128 entries
First usable sector is 34, last usable sector is 314572766
Partitions will be aligned on 2048-sector boundaries
Total free space is 2047 sectors (1023.5 KiB)
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048          411647   200.0 MiB   EF00  EFI System Partition
   2          411648         1435647   500.0 MiB   0700
   3         1435648       266336255   126.3 GiB   8E00
   4       266336256       266338270   1007.5 KiB  8E00  Linux LVM
   5       266338304       314572766   23.0 GiB    8E00  Linux LVM
Command (? for help): w
Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!
Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/sda.
yWarning: The kernel is still using the old partition table.
The new table will be used at the next reboot.
The operation has completed successfully.
```

Теперь же видно, что то самое свободное пространство, которое нам нужно, наконец сформировано в 5 партиции. Ребутим виртуальную машину, после чего начнем работу по расширению пространства. 
Теперь необходимо создать физический том sda5:

```
# pvcreate /dev/sda5
  Physical volume "/dev/sda5" successfully created
```

Далее расширяем группу томов, на новое пространство. Используем наше имя группы томов centos, которое мы подсмотрели ранее, командой df: 

```
# vgextend /dev/centos /dev/sda5
  Volume group "centos" successfully extended
```

Теперь расширим логический том. Вспоминаем, что говорил нам df. 

```
# lvextend -l+100%FREE /dev/centos/root
  Size of logical volume centos/root changed from 124,31 GiB (31823 extents) to 147,31 GiB (37711 extents).
  Logical volume root successfully resized.
```

Еще пару волшебных действий для активации: 

```
# vgscan
  Reading all physical volumes.  This may take a while...
  Found volume group "centos" using metadata type lvm2
# vgchange -ay
  2 logical volume(s) in volume group "centos" now active
```

И последнее, что мы делаем - расширяем файловую систему: 

```
# resize2fs /dev/centos/root
resize2fs 1.42.9 (28-Dec-2013)
Filesystem at /dev/centos/root is mounted on /; on-line resizing required
old_desc_blocks = 16, new_desc_blocks = 19
The filesystem on /dev/centos/root is now 38616064 blocks long.
```

Для CentOS 7 с файловой системой xfs используйте xfs_growfs вместо resize2fs. Данный процесс может занять некоторе время. После завершения операции проверим чего мы натворили: 

```
# df -h
Файловая система        Размер Использовано  Дост Использовано% Cмонтировано в
/dev/mapper/centos-root   145G         2,8G  136G            2% /
devtmpfs                  896M            0  896M            0% /dev
tmpfs                     906M            0  906M            0% /dev/shm
tmpfs                     906M         8,3M  898M            1% /run
tmpfs                     906M            0  906M            0% /sys/fs/cgroup
/dev/sda2                 494M         159M  336M           33% /boot
/dev/sda1                 200M         9,5M  191M            5% /boot/efi
tmpfs                     182M            0  182M            0% /run/user/0
```
