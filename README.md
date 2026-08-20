# Grafana Monitoring Stack

Tek bir `docker compose` ile kurulan sunucu izleme yığını: **Grafana + Prometheus +
Loki**. Sistem metrikleri, site erişilebilirlik kontrolü, nginx trafik analizi ve
SSH giriş logları tek yerde.

Dashboard'lar repoyla birlikte gelir — klonla, `up` de, hazır.

![nginx](img/nginx.png)
![blackbox](img/blackbox.png)

## İçinde ne var

| Servis | Ne yapar | Port |
|---|---|---|
| **grafana** | arayüz | `3000` |
| **prometheus** | metrik toplar | `127.0.0.1:9090` |
| **node-exporter** | cpu / ram / disk / network | `127.0.0.1:9100` |
| **blackbox-exporter** | site up/down, SSL, yanıt süresi | `127.0.0.1:9115` |
| **loki** | log deposu | `127.0.0.1:3100` |
| **promtail** | nginx + auth loglarını Loki'ye taşır | `127.0.0.1:9080` |

Grafana dışındaki her şey `127.0.0.1`'e bağlıdır, dışarıdan erişilemez.

## Hızlı başlangıç

```bash
git clone <repo-url> monitoring && cd monitoring
cp .env.example .env        # şifreyi değiştir
docker compose up -d
```

`http://SUNUCU_IP:3000` → `admin` / `.env`'de belirlediğin şifre.

Datasource'lar (Prometheus + Loki) ve 6 dashboard otomatik yüklenir, elle bir şey
eklemene gerek yok.

> `.env`'deki şifre yalnızca ilk açılışta, `grafana-data` volume'ü boşken uygulanır.
> Sonrasında şifre Grafana'nın kendi veritabanında tutulur.

## Ön koşullar

Yığın **Debian/Ubuntu** varsayar. İki dosya promtail'e bağlanır:

```
/var/log/nginx/access.log    nginx trafik logları
/var/log/auth.log            SSH giriş denemeleri
```

> ⚠️ **Bind mount kaynağı yoksa Docker onu dizin olarak oluşturur.** Promtail hata
> vermez, sadece hiçbir şey okumaz; dashboard'lar boş kalır ve sebebi görünmez.
> Başlamadan önce `ls -la /var/log/auth.log` ile kontrol et.

### SSH logları

| Dağıtım | Yol | Yapılacak |
|---|---|---|
| Debian / Ubuntu | `/var/log/auth.log` | hazır, bir şey yapma |
| RHEL / Rocky / Fedora | `/var/log/secure` | `docker-compose.yml`'deki mount'u ve `promtail/promtail.yaml`'daki `__path__`'i değiştir |
| Sadece journald | dosya yok | `rsyslog` kur, ya da promtail'in `journal` scrape'ine geç |

Bunun dışında ek ayar gerekmez; SSH dashboard'u ham `sshd` satırlarını kendi ayrıştırır.

### Nginx access logları — JSON formatı gerekiyor

Nginx dashboard'u `| json` parser'ı kullanır. **Nginx varsayılan `combined`
formatında yazarsa dashboard tamamen boş gelir.**

```bash
sudo cp nginx/log-json.conf /etc/nginx/conf.d/log-json.conf
# nginx.conf'taki http{} bloğunda mevcut "access_log ..." satırını yorum satırı yap
sudo nginx -t && sudo systemctl reload nginx
```

Doğrula — çıktı JSON olmalı:

```bash
tail -n1 /var/log/nginx/access.log
```

Promtail bu JSON'daki `host` alanını Loki label'ına çevirir; dashboard'un stream
seçicisi buna dayanır. Bu adım atlanırsa paneller "No data" der.

Nginx kullanmıyorsan `promtail/promtail.yaml`'daki `nginx` bloğunu ve
`docker-compose.yml`'deki `/var/log/nginx` mount'unu sil.

## Yapılandırma

### İzlenen siteler

`prometheus.yml` → `blackbox` job'undaki `targets` listesi. Şu an repo sahibinin
adresleri duruyor, kendi domainlerinle değiştir:

```yaml
    static_configs:
      - targets:
          - https://muratakpinar.com.tr
          - https://grafana.muratakpinar.com.tr
```

```bash
docker compose restart prometheus
```

### Yeni log kaynağı

`promtail/promtail.yaml`'a bir `scrape_configs` girdisi ekle, log dosyasını da
`docker-compose.yml`'de promtail'e `:ro` olarak mount et.

## Dashboard'lar

`grafana/dashboards/` altındaki dosyalar açılışta otomatik yüklenir.

| Dashboard | Ne gösterir |
|---|---|
| 1 Linux Stats with Node Exporter | cpu, ram, disk, network |
| Analytics - NGINX | trafik, durum kodları, en çok istenen sayfalar |
| SSH Logs | başarılı/başarısız giriş denemeleri, kaynak IP'ler |
| Promtail Monitoring | log toplayıcının kendi sağlığı |
| Blackbox Exporter (HTTP prober) | site up/down, SSL süresi, yanıt süresi |
| Prometheus Blackbox Exporter | detaylı probe metrikleri |

### Dashboard'ları değiştirmek

Provisioned dashboard'lar arayüzden **kaydedilemez** — Grafana
`Cannot save provisioned dashboard` der. Bu bilinçli: provisioning tek yönlü
çalışır (dosya → Grafana) ve her turda dosyadaki hâli geri yazar, yani arayüzdeki
düzenleme kalıcı olamaz.

İki yolun var:

1. **Kendi kopyanı çıkar** — dashboard'da **Save as copy**. Kopya provisioning'e
   tabi değildir, istediğin gibi düzenlersin, kalıcıdır.
2. **Dosyayı düzenle** — `grafana/dashboards/*.json`. Provisioning 30 saniyede
   bir tarar, `docker compose restart grafana` bile gerekmez.

Yeni dashboard eklemek için JSON'ı `grafana/dashboards/` altına koyman yeterli.

## Dosya yapısı

```
docker-compose.yml
prometheus.yml                     prometheus scrape ayarları
loki/loki-config.yaml
promtail/promtail.yaml             log kaynakları + host label çıkarımı
nginx/log-json.conf                nginx JSON log formatı (kopyalanacak)
grafana/provisioning/datasources/  Prometheus + Loki tanımı
grafana/provisioning/dashboards/   dashboard yükleyici
grafana/dashboards/*.json          dashboard'ların kendisi
```

## Bilinen sınırlar

- **Ülke kırılımı paneli boş kalır.** Nginx dashboard'u `geoip_country_code`
  alanını bekler; bu alan nginx'in `ngx_http_geoip2_module`'ü kurulu değilse
  üretilmez.
- Grafana `3000` portunda dışarı açıktır. Önüne reverse proxy + TLS koy, ya da
  `docker-compose.yml`'de `"127.0.0.1:3000:3000"` yapıp öyle proxy'le.
- Compose proje adı `grafana` olarak sabitlenmiştir (`name: grafana`), böylece
  volume adları dizin adından bağımsızdır.

## Veri kalıcılığı

Her şey adlandırılmış volume'lerde durur: `grafana_grafana-data`,
`grafana_prometheus-data`, `grafana_loki-data`, `grafana_promtail-positions`.

`docker compose down` bunlara dokunmaz. Silmek için `docker compose down -v`.

## Kaynaklar

Dashboard'lar topluluk çalışmalarına dayanıyor:

- [Blackbox Exporter (HTTP prober)](https://grafana.com/grafana/dashboards/13659) — gnetId 13659
- [Prometheus Blackbox Exporter](https://grafana.com/grafana/dashboards/7587) — gnetId 7587
- [Promtail Monitoring](https://grafana.com/grafana/dashboards/20881) — gnetId 20881
- [VoidQuark Grafana Dashboards](https://github.com/voidquark/grafana-dashboards)
- [PrivateBin Access Log](https://grafana.com/grafana/dashboards/19507-privatebin-access-log/)
- [SSH Logs](https://grafana.com/grafana/dashboards/17514-ssh-logs/)

Bu repo iki ayrı projenin birleşimidir:
[blackbox-exporter-docker](https://github.com/murat-akpinar/blackbox-exporter-docker) ve
[nginx-monitoring](https://github.com/murat-akpinar/nginx-monitoring).

## Lisans

[GNU GPL v3](LICENSE)
