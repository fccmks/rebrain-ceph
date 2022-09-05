## Создайте пул для RBD и lun в 5GB и выдайте их через iscsi, предварительно установив iscsi-шлюз

IP-адреса тестовых машин:
- ceph1 - 10.129.0.10
- ceph2 - 10.129.0.20
- ceph3 - 10.129.0.30

### Создайте пул rbd c именем iscsi и в нем lun 5gb
```bash
ceph osd pool create iscsi 32 32
ceph osd pool application enable iscsi rbd
# смотрим список пулов
ceph osd lspools
# создаём lun
rbd create DISK1 --size 5G --pool iscsi
```

### Разверните iscsi gateway (УЗ admin01 passw01 размещение ceph1 ceph2 добавить в доверенные ip и прописать их адреса)
```bash
ceph orch apply iscsi iscsi admin01 passw01 --placement ceph1,ceph2 --trusted_ip_list 10.129.0.10,10.129.0.20,10.129.0.30
ceph orch ls
```

### На первом узле вычислите контейнер iscsi gateway и войдите в него через cephadm enter
```bash
# смотрим имена всех контейнеров
cephadm ls --no-detail | jq '.[].name'
# заходим в нужный
cephadm enter -n iscsi.iscsi.ceph1.fgetan
```

### Используя gwcli, создайте таргет, после чего зацепите шлюзы подключения за ранее созданный диск
```bash
gwcli
cd iscsi-targets
create iqn.2022-09.local.ceph.iscsi-gw:iscsi-igw
cd iqn.2022-09.local.ceph.iscsi-gw:iscsi-igw/gateways
create ceph1.local 10.129.0.10 skipchecks=true
create ceph2.local 10.129.0.20 skipchecks=true
cd /disks
attach iscsi/DISK1
```

### Не выходя из gwcli, создайте клиента с уз (username=iscsiadmin password=iscsipassword) и дайте ему диск
```bash
cd /iscsi-targets/iqn.2022-09.local.ceph.iscsi-gw:iscsi-igw/hosts
create iqn.2022-09.local.client:ubuntu-client
auth username=iscsiadmin password=iscsipassword
disk add iscsi/DISK1
exit
```

### На клиенте поставьте ПО icsci. Настройте клиент на подключение
```bash
apt install open-iscsi multipath-tools
yum install iscsi-initiator-utils device-mapper-multipath
# в раздел CHAP пишем логин и пароль
vi /etc/iscsi/iscsid.conf
# сюда пишем имя нашего пользователя: iqn.2022-09.local.client:ubuntu-client
vi /etc/iscsi/initiatorname.iscsi
```

### Настройте мультипас драйвер
Правим /etc/multipath.conf
```
 devices {
    device {
                 vendor                 "LIO-ORG"
                 product                "TCMU device"
                 hardware_handler       "1 alua"
                 path_grouping_policy   "failover"
                 path_selector          "queue-length 0"
                 failback               60
                 path_checker           tur
                 prio                   alua
                 prio_args              exclusive_pref_bit
                 fast_io_fail_tmo       25
                 no_path_retry          queue
         }
 }
```
### Запустите мультипас и демона icsci
```bash
systemctl enable multipathd && systemctl start multipathd
systemctl enable iscsid && systemctl start iscsid
```

### Отсканируйте и примапьте Lun через iscsi
```bash
# сканируем
iscsiadm -m discovery -t st -p 10.129.0.10
# логинимся
iscsiadm -m node -T iqn.2022-09.local.ceph.iscsi-gw:iscsi-igw -l
# проверяем
multipath -l
lsblk
```

### Создайте директорию /test и смонтируйте в нее диск
```bash
mkdir /test
mkfs.xfs /dev/mapper/360014050e282662bab04091aeb4a87ed
mount /dev/mapper/360014050e282662bab04091aeb4a87ed /test
cp -r /etc/* /test/
```
