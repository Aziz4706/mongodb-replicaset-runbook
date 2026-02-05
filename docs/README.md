# MongoDB Standalone → Replica Set Geçiş Rehberi

Bu doküman MongoDB’nin **Replica Set** mimarisini, neden gerekli olduğunu
ve Docker üzerinde nasıl kurulacağını anlatır.

## Neden Replica Set?

- High Availability
- Otomatik failover
- Veri güvenliği
- Production best practice

## Mimari

Replica set adı: `rs0`

Primary:
- Yazma işlemleri

Secondary:
- Replikasyon
- Okuma yükü (opsiyonel)

Detaylı adımlar için ana README dokümanına bakınız.
