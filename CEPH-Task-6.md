## Создайте RGW и выдайте bucket через s3

IP-адреса тестовых машин:
- ceph1 - 10.129.0.10
- ceph2 - 10.129.0.20
- ceph3 - 10.129.0.30

### Поднимите RGW realm=ceph.local, rgw-zone=ceph.local, все остальное defaults, машины для размещения шлюзов — ceph1 и ceph2
```bash
# создайте реалм
radosgw-admin realm create --rgw-realm=ceph.local --default
# создайте одну зону
radosgw-admin zonegroup create --rgw-zonegroup=default --master --default
# создайте одну группу
radosgw-admin zone create --rgw-zonegroup=default --rgw-zone=ceph.local --master --default
# коммит изменений и разворот шлюзов
radosgw-admin period update --rgw-realm=ceph.local --commit
# размещаем контейнеры
ceph orch apply rgw ceph --realm=ceph.local --zone=ceph.local --placement="2 ceph1 ceph2"
```
### Создайте пользователя test, скопируйте блок с его УЗ на клиентскую машину в файл /root/answer
```bash
radosgw-admin user create --display-name="Ceph Study" --uid=test
```
Формат файла _/root/answer_:
```
"user": "test",
"access_key": "84UL8EYOSWXXAAOIQ23J",
"secret_key": "mCglxWpeMFCIojEkSngO0HjheHVduQTH5sHCQfoO"
```

### На клиенте установите необходимый софт (awscli boto boto3 через pip install)
```bash
apt install python3-pip
pip install awscli boto boto3
```

### Под рутом сконфигурируйте клиент, используя полученные от кластера ключи с профилем ceph (--profile=ceph)
```bash
aws configure --profile=ceph
```

### Под рутом проверьте настройки и доступ к гейтвею. Если все работает, создайте бакет test
```bash
aws --profile=ceph --endpoint=http://ceph1 s3api create-bucket --bucket test
aws --profile=ceph --endpoint=http://ceph1 s3 ls
```

### Создайте файл testfile, запишите в него строку "S3 is Working". Скопируйте файл в бакет test
```bash
echo "S3 is Working" > testfile
aws --profile=ceph --endpoint=http://ceph1 s3 cp testfile s3://test/
aws --profile=ceph --endpoint=http://ceph1 s3 ls s3://test/
```
