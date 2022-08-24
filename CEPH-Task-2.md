## Установите кластер CEPH на трех машинах 
`Для целей обучения OSD и MON установлены на одних и тех же узлах. На проде же OSD живут на своих, как правило, железных серверах, MON на своих, как правило, виртуальных машинах. Это же касается прочих демонов типа iscsi gw. Но на курсе мы ограничены виртуализацией и ресурсами - поэтому совмещаем.`

IP-адреса тестовых машин:
- ceph1 - 10.129.0.10
- ceph2 - 10.129.0.20
- ceph3 - 10.129.0.30
### На первой машине создайте ключ и разошлите на остальные машины:
```bash
ssh-keygen
ssh-copy-id root@ceph1 && ssh-copy-id root@ceph2 && ssh-copy-id root@ceph3
```

### Поставьте ceph-common на первый узел
```bash
yum install ceph-common
```

### Поставьте CEPH через cephadm bootstrap
```bash
mkdir /etc/ceph
cephadm bootstrap --mon-ip 10.129.0.10 --initial-dashboard-user mksuser --initial-dashboard-password mksuser --dashboard-password-noupdate
```

### Скопируйте ключ CEPH на остальные машины
```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph3
```

### Добавьте в кластер узлы и добавьте label узлов (mon, mgr)
```bash
ceph orch host add ceph2 10.129.0.20
ceph orch host add ceph3 10.129.0.30

ceph orch host label add ceph1 mon
ceph orch host label add ceph2 mon
ceph orch host label add ceph3 mon
ceph orch host label add ceph1 mgr
ceph orch host label add ceph2 mgr
ceph orch host label add ceph3 mgr
```

### Сконфигурируйте сеть кластера
```bash
ceph config set mon public_network 10.129.0.0/24
```

### Запустите мониторы (mon, mgr)
```bash
ceph orch apply mon "ceph1,ceph2,ceph3"
ceph orch apply mgr "ceph1,ceph2,ceph3"
```

### Добавьте диски узлов в кластер.
```bash
ceph orch daemon add osd ceph1:/dev/vdb
ceph orch daemon add osd ceph2:/dev/vdb
ceph orch daemon add osd ceph3:/dev/vdb
```

### Проверяем
```bash
ceph orch host ls
ceph osd tree | column -t -s $'\t'
```
