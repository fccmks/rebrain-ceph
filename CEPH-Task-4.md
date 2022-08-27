## Создайте пул для CEPHFS, выдайте FS и смонитируйте ее на клиент

### Создайте пул CEPHFS c именем volume1 и задеплойте MDS
```bash
ceph fs volume create volume1
ceph orch apply mds volume1 3
```

### Проверьте количество PG \ PGP в пуле и измените их на 32
```bash
# смотрим список пулов
ceph osd lspools
ceph osd pool get cephfs.volume1.data pg_num
ceph osd pool get cephfs.volume1.data pgp_num
ceph osd pool get cephfs.volume1.meta pg_num
ceph osd pool get cephfs.volume1.meta pgp_num
# меняем значения только для пула с данными
ceph osd pool set cephfs.volume1.data pg_num 32
ceph osd pool set cephfs.volume1.data pgp_num 32
```

### Сгенерируйте ключ для volume1
```bash
ceph fs authorize volume1 client.user / rw
```

### На клиентах 1 и 2 смонтируйте FS в директорию /test, предварительно создав её
```bash
mkdir /test
mount -t ceph ceph1,ceph2,ceph3:/ /test -o name=user,secret=<ключ>
```

### Создайте на клиенте 1 в директории test файл test с содержимым "This is a node 1"
```bash
echo "This is a node 1" > /test/test
```

### На клиенте 2 допишите в конец файла /test/test "This is a node 2"
```bash
echo "This is a node 2" >> /test/test
```

### Проверяем
```bash
# проверяем содержимое файла на обоих клиентах
cat /test/test
# смотрим состояние autoscale
ceph osd pool autoscale-status
```
