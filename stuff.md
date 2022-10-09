### Проверка производительности OSD
```bash
# выполняется на ноде с тестируемой OSD
ceph tell osd.0 bench
```
### Нагрузочное тестирование rbd диска с помощью fio
```bash
cat /etc/test.fio
[write-4M]
description="write test with block size of 4M"
ioengine=rbd
clientname=admin
pool=rbd
rbdname=disk01
iodepth=32
runtime=120
# write/read is sequential write/read, randwrite/randread is random write/read
rw=write
bs=4M
```