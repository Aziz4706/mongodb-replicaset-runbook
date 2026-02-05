# Troubleshooting

## rs.add() başarısız

- Hostname çözümlemesi kontrol edin
- `/etc/hosts` veya DNS
- Firewall 27017/tcp açık mı?
- Tüm node’larda `--replSet rs0` aynı mı?

## Authentication hataları

- `--authenticationDatabase admin` parametresi verildi mi?
- Keyfile permission `400` mü?
- Keyfile tüm node’larda birebir aynı mı?

## Secondary sürekli RECOVERING

- Disk performansı (IOPS)
- Network latency
- Initial sync devam ediyor olabilir
