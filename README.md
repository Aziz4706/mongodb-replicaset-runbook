# MongoDB Docker Standalone → Replica Set Migration Guide

Bu doküman, Docker üzerinde **standalone (tek node)** çalışan bir MongoDB kurulumunun,
**3 node’lu bir Replica Set (rs0)** mimarisine güvenli ve kontrollü şekilde taşınmasını anlatır.

Doküman hem **operasyonel adımları**, hem de **neden bu adımların gerekli olduğunu**
açıklar. Amaç sadece “çalıştırmak” değil, **doğru ve sürdürülebilir** bir MongoDB cluster kurmaktır.

---

## İçindekiler

- [Replica Set Nedir?](#replica-set-nedir)
- [Neden Standalone Yetmez?](#neden-standalone-yetmez)
- [Replica Set’in Sağladığı Avantajlar](#replica-setin-sagladigi-avantajlar)
- [Mimari Genel Bakış](#mimari-genel-bakis)
- [Ön Gereksinimler](#on-gereksinimler)
- [Adım 1 – Network ve Hostname Çözümleme](#adım-1--network-ve-hostname-cozumleme)
- [Adım 2 – Keyfile (Internal Auth)](#adım-2--keyfile-internal-auth)
- [Adım 3 – Primary Node’u Replica Set Moduna Alma](#adım-3--primary-nodeu-replica-set-moduna-alma)
- [Adım 4 – Replica Set Initialize](#adım-4--replica-set-initialize)
- [Adım 5 – Secondary Node’ları Ayağa Kaldırma](#adım-5--secondary-nodeları-ayaga-kaldırma)
- [Adım 6 – Secondary’leri Cluster’a Ekleme](#adım-6--secondaryleri-clustera-ekleme)
- [Connection String ve Uygulama Tarafı](#connection-string-ve-uygulama-tarafı)
- [Replica Set Priority ve Failover Davranışı](#replica-set-priority-ve-failover-davranısı)
- [Sağlık ve Replikasyon Kontrolleri](#saglık-ve-replikasyon-kontrolleri)
- [Troubleshooting](#troubleshooting)
- [Rollback Plan](#rollback-plan)
- [Production Checklist](#production-checklist)

---

## Replica Set Nedir?

MongoDB Replica Set;

- **1 Primary**
- **1 veya daha fazla Secondary**

node’dan oluşan bir **yüksek erişilebilirlik (High Availability)** yapısıdır.

Primary:
- Yazma işlemlerini kabul eder

Secondary:
- Primary’den **oplog** üzerinden veriyi replikasyon ile alır
- İstenirse okuma yükünü paylaşabilir
- Primary düşerse **otomatik failover** ile yeni primary seçilir

---

## Neden Standalone Yetmez?

Standalone MongoDB:

- ❌ Tek hata noktasıdır (Single Point of Failure)
- ❌ Disk, OS veya container çökmesinde servis tamamen durur
- ❌ Bakım sırasında downtime zorunludur
- ❌ Ölçeklenebilirlik ve okuma dağıtımı yoktur

---

## Replica Set’in Sağladığı Avantajlar

### ✅ High Availability
- Primary düşerse otomatik olarak secondary’lerden biri primary olur

### ✅ Veri Güvenliği
- Veriler birden fazla node’da tutulur
- Disk veya node kaybında veri kaybı yaşanmaz (doğru yapılandırmada)

### ✅ Bakım Kolaylığı
- Secondary node’lar sırayla bakıma alınabilir
- Versiyon upgrade’leri daha kontrollü yapılır

### ✅ Okuma Ölçeklenebilirliği
- `readPreference=secondary` ile okuma yükü dağıtılabilir

### ✅ Production Best Practice
- MongoDB’nin **önerdiği ve desteklediği** üretim mimarisidir

---

## Mimari Genel Bakış

```
        ┌───────────┐
        │  mongo1   │
        │  PRIMARY  │
        └─────┬─────┘
              │ oplog
   ┌──────────┴──────────┐
┌──▼──────┐        ┌─────▼─────┐
│ mongo2  │        │  mongo3   │
│SECONDARY│        │ SECONDARY │
└─────────┘        └───────────┘
```

Replica Set adı: `rs0`

---

## Ön Gereksinimler

- Docker & Docker Compose
- MongoDB 6.x (örnek: `mongo:6.0.27`)
- Node’lar arası TCP/27017 erişimi
- Zaman senkronizasyonu (NTP önerilir)
- Kalıcı disk (volume) kullanımı

> Not: Bu doküman örnek olarak `acme` isimli anonim path/container isimleri kullanır.
> Kendi ortamınıza göre değiştirin.

---

## Adım 1 – Network ve Hostname Çözümleme

Replica set üyeleri **birbirini hostname ile görmelidir**.

IP kullanımı mümkündür ancak:
- IP değişirse `rs.conf()` içindeki host mapping bozulur
- Failover / reconfig sırasında sorun çıkar

### `/etc/hosts`

Her node’a (mongo1/mongo2/mongo3) aynı kaydı girin:

```bash
sudo bash -lc 'cat >> /etc/hosts <<EOF
10.10.10.11 mongo1
10.10.10.12 mongo2
10.10.10.13 mongo3
EOF'
```

Test:

```bash
ping -c 1 mongo1
ping -c 1 mongo2
ping -c 1 mongo3
```

### Firewall
Node’lar arası **27017/tcp** açık olmalı.

---

## Adım 2 – Keyfile (Internal Auth)

Replica set node’ları **birbirine authentication yapar**.
Bu auth **keyfile** ile sağlanır.

Keyfile olmadan:
- `--auth` + `--replSet` birlikte sağlıklı çalışmaz
- Node’lar birbirine güvenmez, replikasyon kopar

Keyfile:
- Tüm node’larda **aynı içerik**
- Permission: `400`
- Owner: MongoDB user (official image genelde `999:999`)

### 2.1 mongo1 üzerinde keyfile oluştur

```bash
sudo mkdir -p /opt/acme/mongodb/security
sudo openssl rand -base64 756 | sudo tee /opt/acme/mongodb/security/keyfile >/dev/null
sudo chmod 400 /opt/acme/mongodb/security/keyfile
sudo chown 999:999 /opt/acme/mongodb/security/keyfile
```

### 2.2 Keyfile’ı mongo2 ve mongo3’e kopyala

mongo2:

```bash
sudo scp /opt/acme/mongodb/security/keyfile root@mongo2:/opt/acme/mongodb/security/keyfile
sudo ssh root@mongo2 'chmod 400 /opt/acme/mongodb/security/keyfile && chown 999:999 /opt/acme/mongodb/security/keyfile'
```

mongo3:

```bash
sudo scp /opt/acme/mongodb/security/keyfile root@mongo3:/opt/acme/mongodb/security/keyfile
sudo ssh root@mongo3 'chmod 400 /opt/acme/mongodb/security/keyfile && chown 999:999 /opt/acme/mongodb/security/keyfile'
```

---

## Adım 3 – Primary Node’u Replica Set Moduna Alma

Bu adımda mevcut standalone MongoDB, replica set aware hale getirilir.

Eklenecek flag’ler:
- `--replSet rs0`
- `--keyFile /data/keyfile`
- `--bind_ip_all`
- `--auth`

⚠️ **Kısa downtime burada oluşur** (container restart).

> Önemli: `MONGO_INITDB_*` env’leri yalnızca **ilk init** sırasında çalışır.
> Data volume doluysa bu env’lere güvenmeyin; kullanıcılar zaten DB’de durur.

### 3.1 mongo1 docker-compose.yml örneği

```yaml
version: '3.4'
services:
  mongodb:
    image: mongo:6.0.27
    restart: always
    container_name: mongo-rs-mongo1
    hostname: mongo1
    environment:
      - MONGO_INITDB_DATABASE=admin
      - MONGO_INITDB_ROOT_USERNAME=mongopassword
      - MONGO_INITDB_ROOT_PASSWORD=mongoadmin
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /opt/acme/mongodb/db/:/data/db
      - /opt/acme/mongodb/initdb.d/init-mongo.sh:/docker-entrypoint-initdb.d/init-mongo.sh:ro
      - /opt/acme/mongodb/log/:/var/log/mongodb/
      - /opt/acme/mongodb/security/keyfile:/data/keyfile:ro
    ports:
      - "27017:27017"
    command: ["mongod","--replSet","rs0","--keyFile","/data/keyfile","--bind_ip_all","--auth"]
```

**Notlar**
- `hostname: mongo1` replset içinde doğru isimle görünmesi için.
- `command` kullanımı entrypoint’i bozmaz; init davranışı korunur.
- `--auth`: Eğer bazı servisler “auth’suz” bağlanıyorsa bu adım onları kırar; o durumda geçici olarak `--auth` kaldırıp sonra kontrollü aktive edin.

### 3.2 Restart (downtime)

```bash
docker compose down
docker compose up -d
docker logs -f mongo-rs-mongo1
```

Kontrol:

```bash
docker exec -it mongo-rs-mongo1 mongosh -u mongoadmin -p mongopassword --authenticationDatabase admin --eval 'db.runCommand({ping:1})'
```

---

## Adım 4 – Replica Set Initialize

Replica set kendiliğinden oluşmaz, manuel başlatılır.

mongo1 shell:

```bash
docker exec -it mongo-rs-mongo1 mongosh -u mongoadmin -p mongopassword --authenticationDatabase admin
```

Initialize:

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017" }
  ]
})
```

Kontrol:

```javascript
rs.status()
```

Beklenen: `mongo1` PRIMARY.

---

## Adım 5 – Secondary Node’ları Ayağa Kaldırma

Secondary node’lar boş data dizini ile başlar, primary’den initial sync yapar.

> Initial sync süresi veri boyutuna ve network hızına bağlıdır.

### 5.1 mongo2 docker-compose.yml örneği

```yaml
version: '3.4'
services:
  mongodb:
    image: mongo:6.0.27
    restart: always
    container_name: mongo-rs-mongo2
    hostname: mongo2
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /opt/acme/mongodb/db/:/data/db
      - /opt/acme/mongodb/security/keyfile:/data/keyfile:ro
    ports:
      - "27017:27017"
    command: ["mongod","--replSet","rs0","--keyFile","/data/keyfile","--bind_ip_all","--auth"]
```

Başlat:

```bash
docker compose up -d
docker exec -it mongo-rs-mongo2 mongosh --eval 'db.runCommand({ping:1})'
```

### 5.2 mongo3 docker-compose.yml örneği

```yaml
version: '3.4'
services:
  mongodb:
    image: mongo:6.0.27
    restart: always
    container_name: mongo-rs-mongo3
    hostname: mongo3
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /opt/acme/mongodb/db/:/data/db
      - /opt/acme/mongodb/security/keyfile:/data/keyfile:ro
    ports:
      - "27017:27017"
    command: ["mongod","--replSet","rs0","--keyFile","/data/keyfile","--bind_ip_all","--auth"]
```

Başlat:

```bash
docker compose up -d
docker exec -it mongo-rs-mongo3 mongosh --eval 'db.runCommand({ping:1})'
```

---

## Adım 6 – Secondary’leri Cluster’a Ekleme

mongo1 shell’e girin:

```bash
docker exec -it mongo-rs-mongo1 mongosh -u mongoadmin -p mongopassword --authenticationDatabase admin
```

Ekleyin:

```javascript
rs.add("mongo2:27017")
rs.add("mongo3:27017")
```

Kontrol:

```javascript
rs.status()
```

Beklenen:
- mongo1 PRIMARY
- mongo2 SECONDARY
- mongo3 SECONDARY

---

## Connection String ve Uygulama Tarafı

Temel:

```text
mongodb://mongoadmin:mongoadmin@mongo1:27017,mongo2:27017,mongo3:27017/admin?replicaSet=rs0&authSource=admin
```

Okuma dağıtımı (opsiyonel):

```text
&readPreference=secondaryPreferred
```

> Not: Yazmalar her zaman primary’ye gider. ReadPreference yalnızca okuma routing’i etkiler.

---

## Replica Set Priority ve Failover Davranışı

Priority, hangi node’un primary olmaya “daha istekli” olduğunu belirler.

Örn: mongo1 tercih edilen primary olsun:

```bash
docker exec -it mongo-rs-mongo1 mongosh -u mongoadmin -p mongoadmin --authenticationDatabase admin --eval '
cfg = rs.conf();
cfg.members[0].priority = 2; // mongo1
cfg.members[1].priority = 1; // mongo2
cfg.members[2].priority = 1; // mongo3
rs.reconfig(cfg);
'
```

Priority görüntüleme:

```bash
docker exec -it mongo-rs-mongo1 mongosh -u mongoadmin -p mongopassword --authenticationDatabase admin --eval 'rs.conf().members.map(m=>({host:m.host,priority:m.priority,votes:m.votes}))'
```

---

## Sağlık ve Replikasyon Kontrolleri

Replica set durum:

```bash
docker exec -it mongo-rs-mongo1 mongosh -u mongoadmin -p mongopassword --authenticationDatabase admin --eval 'rs.status()'
```

Health özet:

```bash
docker exec -it mongo-rs-mongo1 mongosh -u mongoadmin -p mongopassword --authenticationDatabase admin --eval 'db.adminCommand({replSetGetStatus:1}).members.map(m=>({name:m.name,stateStr:m.stateStr,health:m.health,syncingTo:m.syncingTo,info:m.info}))'
```

Replication lag:

```bash
docker exec -it mongo-rs-mongo1 mongosh -u mongoadmin -p mongopassword --authenticationDatabase admin --eval 'rs.printSecondaryReplicationInfo()'
```

Aktif connection sayısı:

```bash
docker exec -it mongo-rs-mongo1 mongosh -u mongoadmin -p mongopassword --authenticationDatabase admin --quiet --eval 'db.serverStatus().connections'
```

---
## Troubleshooting

### 1) `rs.add(...)` hata veriyor / node SECONDARY olmuyor
- `/etc/hosts` çözümlemesi doğru mu? (mongo1 mongo2’yi hostname ile çözebiliyor mu?)
- Firewall 27017 açık mı?
- `--replSet rs0` tüm node’larda aynı mı?
- Keyfile tüm node’larda aynı mı ve permission `400` mü?
- Container içinden hostname resolve oluyor mu?

Container içinden test:

```bash
docker exec -it mongo-rs-mongo1 bash -lc 'getent hosts mongo2 && getent hosts mongo3'
```

### 2) `not authorized` / auth hataları
- `--auth` aktifse, `mongosh` bağlanırken `--authenticationDatabase admin` verildi mi?
- Root user gerçekten var mı? (İlk init sırasında oluşmuş olmalı; data volume doluysa env var’lar yeniden çalışmaz.)

### 3) Keyfile permission hatası
- Permission `400` değilse Mongo başlatmaz.
- Owner genelde `999:999` olmalı.

---

## Rollback Plan

> Rollback = “Replica set’i kaldırıp tekrar standalone’a dönmek”.

**Sert gerçek:** Standalone’a dönüş teknik olarak mümkün ama prod ortamda genelde tercih edilmez.
Yine de “acil kaçış” için en basit yaklaşım:

1. Uygulamaları durdur (write işlemlerini kes).
2. Primary node’da verinin güncel olduğundan emin ol:
   - `rs.status()` ile primary’nin sağlıklı olduğunu doğrula.
3. Secondary’leri tamamen kapat (compose down).
4. Primary compose dosyasında `--replSet`, `--keyFile`, `--auth` parametrelerini kaldır.
5. Primary’yi standalone olarak kaldır/kur.
6. Uygulamaları standalone connection string’e döndür.

⚠️ Not: Replica set metadata’sı data dizininde kalacağı için bazı senaryolarda temiz başlangıç gerekebilir.
Rollback gereksinimi varsa üretim öncesi mutlaka staging’de prova edin.

---

## Production Checklist

- [ ] Hostname çözümleme net (DNS veya /etc/hosts)
- [ ] 27017/tcp node’lar arası açık
- [ ] Disk IOPS yeterli (Mongo disk sever)
- [ ] NTP aktif (clock drift seçimleri bozar)
- [ ] Keyfile güvenli path ve doğru permission
- [ ] `--auth` ve güçlü kullanıcı şifreleri
- [ ] Backup stratejisi (mongodump değilse snapshot planı)
- [ ] Monitoring / alerting (replication lag, connections, disk, CPU)
- [ ] Uygulama connection string’inde `replicaSet=rs0` var
- [ ] Failover senaryosu test edildi (primary down → yeni primary)

---


