# Rollback Planı

Replica Set’ten Standalone MongoDB’ye dönüş **önerilmez** ancak
acil durumlar için aşağıdaki yaklaşım izlenebilir.

## Basit Rollback Adımları

1. Uygulamaları durdur
2. Secondary node’ları kapat
3. Primary node’da:
   - `--replSet`
   - `--keyFile`
   - `--auth`
   parametrelerini kaldır
4. MongoDB’yi standalone olarak başlat
5. Uygulama connection string’ini güncelle

⚠️ Not: Replica set metadata’sı disk üzerinde kalabilir.
Staging ortamında mutlaka test edilmelidir.
