## Создайте пул для RBD, выдайте диск и смонитируйте его на клиент
`На клинете обязательно должен быть установлен пакет ceph-common`

### Создайте пул rbd (размер 32 pg, фактор репликации по умолчанию)
```bash
ceph osd pool create rbd 32 32
ceph osd pool application enable rbd rbd
# не обязательно
rbd pool init -p rbd
# смотрим список пулов
ceph osd lspools
```

### Создайте диск (lun rbd) с именем disk01 размером 2GB
```bash
rbd create disk01 --size 2G --pool rbd
# проверяем
rbd --image disk01 -p rbd info
```

### Сгенерируйте минимальный конфиг и пользовательский ключ, затем скопируйте ключ на клиент
```bash
ceph config generate-minimal-conf > ceph.conf
scp ceph.conf client:/etc/ceph/ceph.conf
ceph auth get-or-create client.rbd mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=rbd'
ceph auth get-or-create client.rbd | ssh root@client tee /etc/ceph/keyring
```

### На клиенте смапьте диск (rbd map)
```bash
# проверяем, что клиенту доступны ресурсы
ceph -s --name client.rbd
rbd ls --name client.rbd
# маунтим диск
rbd map --image disk01 --name client.rbd
# проверяем (должен появиться диск rbd0)
lsblk
```

### Подготовьте диск через LVM (vg=test lv=test size=1G)
`По умолчанию LVM не понимает диски типа rbd. Для включения нужно в файле /etc/lvm/lvm.conf добавить строку types = [ “rbd”, 1024 ] в разделе devices.`
```bash
pvcreate /dev/rbd0
vgcreate test /dev/rbd0
lvcreate -n test -L 1G test
```

### Смонтируйте диск в директорию /test как xfs, предварительно создав директорию
```bash
mkdir /test
mkfs.xfs /dev/test/test
mount /dev/test/test /test
```

### Создайте на диске в директории test файл test с содержимым "This is an RBD - it's working"
```bash
vi /test/test
```
