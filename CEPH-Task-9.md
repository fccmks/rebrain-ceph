## Проведите ряд тестов кластера ceph, измените параметры ядра на машинах кластера и повторите замер. Все замеры записывайте в файлы, указанные в заданиях.

## Для выполнения задания необходимо сделать следующие шаги

### Создайте пул rbd и диск на 10 гб. Установите параметр пула size 3 pg 8 и скопируйте ключ на клиент
```bash
# создаём пул
ceph osd pool create rbd 8 8
# вырубаем autoscale
ceph osd pool set rbd pg_autoscale_mode off
# проверяем, что автоскейлер выключен
ceph osd pool autoscale-status
POOL    SIZE  TARGET SIZE  RATE  RAW CAPACITY   RATIO  TARGET RATIO  EFFECTIVE RATIO  BIAS  PG_NUM  NEW PG_NUM  AUTOSCALE  BULK
.mgr  448.5k                3.0        61428M  0.0000                                  1.0       1              on         False
rbd    4106M                3.0        61428M  0.2006                                  1.0       8          32  off        False

# проверяем, что pg у нас 8
ceph osd pool get rbd pg_num
ceph osd pool get rbd pgp_num
# если автоскейлер успел что-то сделать, то возвращаем всё назад
ceph osd pool set rbd pg_num 8
ceph osd pool set rbd pgp_num 8
ceph osd pool set rbd size 3
ceph osd pool application enable rbd rbd
rbd create disk01 --size 10G --pool rbd
# проверяем
rbd --image disk01 -p rbd info
# генерим базовый конфиг и копируем его на клиента
ceph config generate-minimal-conf > ceph.conf
scp ceph.conf client:/etc/ceph/ceph.conf
# в процессе выполнения столкнулся с тем, что ключ был создан, но прав клиенту не хватало,
# поэтому удалил ключ и сделал его заново
ceph auth del client.rbd
ceph auth get-or-create client.rbd mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=rbd'
ceph auth get-or-create client.rbd | ssh root@client tee /etc/ceph/keyring
```
## Проведите тесты на ceph1 и запишите результаты в файл test№(1-3):
* с кластера запись --no-cleanup 10 циклов (цель: Average IOPS)
```bash
# запускаем тест и сохраняем результат
rados bench -p rbd 10 write --no-cleanup &> /root/test1
```
* с кластера чтение последовательное 10 циклов (цель: Average IOPS)
```bash
# запускаем тест и сохраняем результат
rados bench -p rbd 10 seq &> /root/test2
```
* с кластера чтение случайное 10 циклов (цель: Average IOPS)
```bash
# запускаем тест и сохраняем результат
rados bench -p rbd 10 rand &> /root/test3
```
## Очистите пул от тестовых данных
```bash
rados -p rbd cleanup
```
### На клиенте проведите тест записи стандартными блоками --io-total 1719973000 (цель: ops/sec: bytes/sec:) - вывод &> в /root/test1 (rbd bench-write)
```bash
rbd bench-write disk01 --io-total 1719973000 --name client.rbd &> /root/test1
```
### Смонтируйте диск в директорию /test (фс xfs), запустите тест записи dd на файл в ней bs=1G count=4 oflag=direct (цель: скорость записи в mb/sec) - вывод &> в /root/test2 (dd)
```bash
# проверяем, что клиенту доступны ресурсы
ceph -s --name client.rbd
rbd ls --name client.rbd
# маунтим диск
rbd map --image disk01 --name client.rbd
# проверяем (должен появиться диск rbd0)
lsblk
# монтируем диск
mkdir /test
mkfs.xfs /dev/rbd0
mount /dev/rbd0 /test
# запускаем dd
dd if=/dev/zero of=/test/deleteme bs=1G count=4 oflag=direct &> /root/test2
```
### Прочитайте созданный файл c помощью dd bs=1G count=4 iflag=direct (цель: скорость чтения в mb/sec) - вывод &> в /root/test3 (dd)
```bash
dd if=/test/deleteme of=/dev/zero bs=1G count=4 iflag=direct &> /root/test3
```
## На всех узлах кластера примените настройки
```bash
echo 1048576 > /proc/sys/fs/aio-max-nr # или с помощью sysctl fs.aio-max-nr=1048576
echo "8192" > /sys/block/vdb/queue/read_ahead_kb
```
### На кластере выставьте пулу большее количество PG до 32
```bash
ceph osd pool set rbd pg_num 32
ceph osd pool set rbd pgp_num 32
```
### Подождите минуту и повторите все измерения, занося результаты в файлы /root/newtest1, /root/newtest2, /root/newtest3