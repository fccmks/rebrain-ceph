## Установите кластер CEPH на трех машинах 
`OSD и MON должны быть на одних и тех же узлах`
### На первой машине создайте ключ и разошлите на остальные машины:
```bash
ssh-keygen
ssh-copy-id root@ceph1 && ssh-copy-id root@ceph2 && ssh-copy-id root@ceph3
```

### Поставьте ceph-common на первый узел
```bash
yum install ceps-common
```

### Поставьте CEPH через cephadm bootstrap
```bash
mkdir /etc/ceph
cephadm bootstrap --mon-ip 10.129.0.20 --initial-dashboard-user mksuser --initial-dashboard-password mksuser --dasboard-password-noupdate
```

### Скопируйте ключ CEPH на остальные машины
```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph3
```

### Добавьте в кластер узлы и добавьте label узлов (mon, mgr)
```bash
ceph orch host add ceph2 192.168.122.102
ceph orch host add ceph3 192.168.122.103

ceph orch host label add ceph1 mon
ceph orch host label add ceph2 mon
ceph orch host label add ceph3 mon
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
