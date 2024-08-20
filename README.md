### OTUS Linux Professional Lesson #8
#### ЦЕЛЬ:
Знакомство с файловой системой ZFS.
#### ОПИСАНИЕ ДОМАШНЕГО ЗАДАНИЯ:
1. Определить алгоритм с наилучшим сжатием:
- определить какие алгоритмы сжатия поддерживает zfs (gzip, zle, lzjb, lz4)
- создать 4 файловых системы, на каждой применить свой алгоритм сжатия
- для сжатия использовать либо текстовый файл, либо группу файлов
2. Определить настройки пула.
- с помощью команды zfs import собрать pool ZFS
- Командами zfs определить настройки:
    - размер хранилища
    - тип pool
    - значение recordsize
    - какое сжатие используется
    - какая контрольная сумма используется
3. Работа со снапшотами:
- скопировать файл из удаленной директории
- восстановить файл локально. zfs receive
- найти зашифрованное сообщение в файле secret_message

#### ВЫПОЛНЕНИЕ:
Разворачиваем виртуальную машину с 8 дополнительными дисками (vagrant up)
##### ЗАДАНИЕ 1. Определение алгоритма с наилучшим сжатием
1. Смотрим список всех дисков, которые есть в виртуальной машине:
```   
lsblk
```
2. Создаём 4 пула по два диска в каждом в режиме RAID 1:
```
[root@zfs ~]# zpool create otus1 mirror sdb sdc
[root@zfs ~]# zpool create otus2 mirror sdd sde
[root@zfs ~]# zpool create otus3 mirror sdf sdg
[root@zfs ~]# zpool create otus4 mirror sdh sdi
```
Команды для просмотра информацию о пулах:
   
`zpool list` - показывает информацию о размере пула, количеству занятого и свободного места, дедупликации и т.д.
   
`zpool status` - показывает информацию о каждом диске, состоянии сканирования и об ошибках чтения, записи и совпадения
3. Cоздаем дополнительные dataset в каждом пуле или также можно использовать весь пул, так называемый корневой dataset
>[!NOTE]
> В некоторых случая нужно отключить монтирование корневого dataset, чтобы он не мешал при просмотре и чтобы не записать в него что-то случайно:
>   
> `[root@zfs ~]# zfs set mountpoint=none poolname`,  или с опцией `"-m none"` при создании пула командой `zpool create` 
> 
> Соответственно, если мы дальше создадим dataset на таком скрытом пуле, то нужно обязательно указать для него точку монтирования. Например:
> ```
> [root@zfs ~]# zfs set mountpoint=none otus1
> [root@zfs ~]# zfs set mountpoint=/mnt otus1/dataset1
> ```
Cоздаем дополнительные dataset's (разделы) в каждом пуле:
```
zfs create otus1/fs1
zfs create otus2/fs2
zfs create otus3/fs3
zfs create otus4/fs4
```
4. Добавим разные алгоритмы сжатия на каждый dataset:
```
zfs set compression=zle otus1/fs1
zfs set compression=gzip-9 otus2/fs2
zfs set compression=lz4 otus3/fs3
zfs set compression=lzjb otus4/fs4
```
Этой командой проверим, что все dataset's имеют разные методы сжатия:
```
zfs get all | grep compression
```
> [!NOTE]
> Сжатие файлов будет работать только с файлами, которые были добавлены после включение настройки сжатия.
5. Скачаем один и тот же текстовый файл во все пулы:
```
for i in {1..4}; do wget -P /otus$i/fs$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
```
Проверим, что файл был скачан во все пулы:
```
ls -l /otus*/fs*

total 40136
-rw-r--r--. 1 root root 41070955 Aug  2 07:54 pg2600.converter.log

/otus2/fs2:
total 10963
-rw-r--r--. 1 root root 41070955 Aug  2 07:54 pg2600.converter.log

/otus3/fs3:
total 18001
-rw-r--r--. 1 root root 41070955 Aug  2 07:54 pg2600.converter.log

/otus4/fs4:
total 22084
-rw-r--r--. 1 root root 41070955 Aug  2 07:54 pg2600.converter.log

```
Уже на этом этапе видно, что самый оптимальный метод сжатия у нас используется в датасете otus2/fs2.
   
Проверим, сколько места занимает один и тот же файл в разных пулах и проверим степень сжатия файлов:
```
zfs list
NAME        USED  AVAIL     REFER  MOUNTPOINT
otus1      39.3M   313M     25.5K  /otus1
otus1/fs1  39.2M   313M     39.2M  /otus1/fs1
otus2      10.8M   341M     25.5K  /otus2
otus2/fs2  10.7M   341M     10.7M  /otus2/fs2
otus3      17.7M   334M     25.5K  /otus3
otus3/fs3  17.6M   334M     17.6M  /otus3/fs3
otus4      21.7M   330M     25.5K  /otus4
otus4/fs4  21.6M   330M     21.6M  /otus4/fs4
```
```
[root@zfs ~]# zfs get all |grep compressratio |grep -v ref
otus1           compressratio         1.82x                  -
otus1/dataset1  compressratio         1.82x                  -
otus2           compressratio         2.23x                  -
otus2/dataset2  compressratio         2.23x                  -
otus3           compressratio         3.66x                  -
otus3/dataset3  compressratio         3.67x                  -
otus4           compressratio         1.00x                  -
otus4/dataset4  compressratio         1.00x                  -

```
Видим, что алгоритм gzip-9 самый эффективный по сжатию, отсюда можно предположить, что он будет и самый медленный. Оптимальный алгоритм по умолчанию - **lz4**

##### ЗАДАНИЕ 2. Определение настроек пула
1. Скачиваем архив, который содержит экспортированный пул, в домашний каталог:
```
cd ~
wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download'
```
```
tar -xzvf archive.tar.gz
```
2. В качестве устройств данного пула были выбраны файлы, такая архитектура используется для тестирования функции zfs, потому что в таком случае мы будем зависить еще от файловой системы, на которой лежат эти файлы. 
Проверим, возможно ли импортировать к нам этот бэкап:
```
zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

	otus                         ONLINE
	  mirror-0                   ONLINE
	    /root/zpoolexport/filea  ONLINE
	    /root/zpoolexport/fileb  ONLINE

```
Данный вывод показывает нам имя пула, тип raid и его состав.
Команда `zpool import` ищет устройства для импортирования. Ключ -d говорит искать в указанном каталоге. Если не указать этого, `zpool import` будет искать устройства в /dev по умолчанию. 
> [!NOTE]
> Экспорт пула выполняется командой `zpool export tank`, zfs скидывает незаписанные данные с кеша на диск, и отмонтирует пул. Дальше
> можно перенести диски на другой сервер. На данном сервере запускаем `zpool import`. ZFS начинает сканировать блочные устройства в каталоге /dev и выдает нам найденные для импортирования. Далее выполняем импортирование пула.

3. Сделаем импорт данного пула к нам в ОС:
```
zpool import -d zpoolexport/ otus
zpool status
  pool: otus
 state: ONLINE
  scan: none requested
config:

	NAME                         STATE     READ WRITE CKSUM
	otus                         ONLINE       0     0     0
	  mirror-0                   ONLINE       0     0     0
	    /root/zpoolexport/filea  ONLINE       0     0     0
	    /root/zpoolexport/fileb  ONLINE       0     0     0
```
Посмотреть настройки импортированного пула:
```
zpool get all otus
```
Можно также указывать конкретные параметры. Например:
```
zfs get available otus
zfs get readonly otus
zfs get recordsize otus
zfs get compression otus
zfs get checksum otus

```
#### ЗАДАНИЕ 3. Работа со снапшотами, поиск сообщения от преподавателя

1. Скачиваем снапшот:
```
wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download

```
2. Восстанавливаем файловую систему со снапшота:

После выполения этой команды будет создан снапшот и новый dataset.
```
zfs receive otus/test@today < otus_task2.file
```
>[!NOTE]
> `zfs receive` создает снапшот, содержимое которого помещается в поток, направляемый на стандартный ввод. Если получен полный поток, то также создается файловая система. В нашем случае мы получаем поток с файла
> otus_task2.file. Этот файл был создан командой `zfs send`, которая создает потоковое представление снапшота и перенаправялет его на стандартный вывод. По умолчанию, zfs send генерирует полный поток.
> В "otus/test@today" today это имя снашота

Команды для просмотра списка снапшотов:
```
zfs list -t snapshot

zpool get listsnapshots otus
```

3. Ищем в каталоге /otus/test файл с именем “secret_message”:
```
find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message

cat /otus/test/task1/file_mess/secret_message
https://otus.ru/lessons/linux-hl/

```
Видим ссылку на курс OTUS

#### Troubleshooting.
При получении ошибки permission denied (publickey). Выполните команду для отключения использования системных SHH утилит:
```
set VAGRANT_PREFER_SYSTEM_BIN=0
```
Для обновления списка репозиториев для Centos 7 выполните команду:
```
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
```