# 🎓 Yüksek Erişilebilir (HA) Uzaktan Eğitim Altyapısı

Binlerce öğrenciye aynı anda hizmet verebilmek için tasarlanmış, **Yüksek Erişilebilirlik (HA)** ve **Ölçeklenebilirlik** prensiplerine dayalı kapsamlı bir Moodle altyapısının kurulumu rehberidir.

---

## 📋 İçindekiler
1. [Sistem Mimarisi](#sistem-mimarisi)
2. [Bölüm 1: Moodle Uygulama Sunucularının Kurulumu](#bölüm-1-moodle-uygulama-sunucularının-kurulumu)
3. [Bölüm 2: HAProxy ile Yük Dengeleme](#bölüm-2-haproxy-ile-yük-dengeleme)
4. [Bölüm 3: PostgreSQL Kurulumu ve PgBouncer ile SQL Yük Dengeleme](#bölüm-3-postgresql-kurulumu-ve-pgbouncer-ile-sql-yük-dengeleme)
5. [Bölüm 4: Scalelite ve BigBlueButton Kümesi](#bölüm-4-scalelite-ve-bigbluebutton-kümesi)
6. [Sorun Giderme İpuçları](#sorun-giderme-ipuçları)

---

## 🏗️ Sistem Mimarisi

```
┌─────────────────────────────────────────────────────────┐
│ İnternet (Kullanıcılar)                                 │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTPS (443)
                       ▼
        ┌──────────────────────────────┐
        │  HAProxy (Yük Dengeleyici)   │ SSL Şifrelemesi
        │  192.168.1.10                │ Trafik Dağıtımı
        └──────────────────────────────┘
                  │ HTTP (80)
     ┌────────────┼────────────┐
     ▼            ▼            ▼
  ┌──────┐    ┌──────┐    ┌──────┐
  │Node1 │    │Node2 │    │Node10│  ... 10x Moodle Node
  │Nginx │    │Nginx │    │Nginx │    (192.168.1.11-20)
  │PHP   │    │PHP   │    │PHP   │
  └──────┘    └──────┘    └──────┘
     │            │            │
     └────────────┼────────────┘
                  │
        ┌─────────┼──────────┐
        ▼         ▼          ▼
    ┌──────────┐ ┌────────┐ ┌────────┐
    │PgBouncer │ │ Redis  │ │  NFS   │
    │(SQL Bal.)│ │Session │ │Dosyalar│
    │192.168.1.│ │192.168.│ │192.168.│
    │101 :6432 │ │1.185   │ │1.50    │
    └──────────┘ └────────┘ └────────┘
         │
    ┌────┴──────────┐
    ▼               ▼
┌──────────┐  ┌──────────┐
│PostgreSQL│  │PostgreSQL│  (Primary + Replica)
│Primary   │  │Replica   │
│192.168.  │  │192.168.  │
│1.100     │  │1.101     │
└──────────┘  └──────────┘

┌─────────────────────────────────────────┐
│ Canlı Dersler (BigBlueButton Kümesi)   │
├─────────────────────────────────────────┤
│  Scalelite Yük Dengeleyici              │
│  (192.168.1.60)                         │
│         │                               │
│    ┌────┴────┬─────────┐               │
│    ▼         ▼         ▼               │
│  BBB-1    BBB-2    BBB-N               │
└─────────────────────────────────────────┘
```

### 🖥️ Bileşen Özeti

| Bileşen | Sayı | Görev | IP Örneği |
|---------|------|-------|----------|
| **HAProxy** | 1 | SSL Sonlandırması, Trafik Dağıtımı | 192.168.1.10 |
| **Moodle Node** | 10 | Eğitim Platformu (Nginx + PHP) | 192.168.1.11-20 |
| **PostgreSQL Primary** | 1 | Ana Veritabanı (Yazma) | 192.168.1.100 |
| **PostgreSQL Replica** | 1 | Okuma Replikası (Opsiyonel) | 192.168.1.101 |
| **PgBouncer** | 1 | SQL Bağlantı Havuzu / Yük Dengeleme | 192.168.1.100 :6432 |
| **Redis** | 1 | Oturum ve Önbellek | 192.168.1.185 |
| **NFS** | 1 | Ortak Dosya Depolama | 192.168.1.50 |
| **Scalelite** | 1 | BBB Yük Dengeleyici | 192.168.1.60 |
| **BigBlueButton** | N | Canlı Ders Sunucuları | 192.168.1.70+ |

---

## 📌 Bölüm 1: Moodle Uygulama Sunucularının Kurulumu

Bu bölümde, arkasında HAProxy'nin olacağı 10 adet Moodle sunucusundan **ilkini (Node-1)** kuracağız. Ardından konfigürasyon diğer sunuculara kopyalanabilir.

### ⚠️ Ön Koşullar
- Ubuntu 20.04+ veya Debian sunucusu
- Root veya sudo yetkisi
- Ağda erişilebilir PostgreSQL sunucusu
- Ağda erişilebilir Redis sunucusu
- Ağda erişilebilir NFS sunucusu

---

### 1️⃣ Gerekli Yazılımların Kurulması

```bash
# Sistem paketlerini güncelle
sudo apt update && sudo apt upgrade -y

# Nginx, PHP 8.3 ve gerekli eklentileri kur
sudo apt install -y \
  nginx \
  php8.3-fpm \
  php8.3-pgsql \
  php8.3-curl \
  php8.3-gd \
  php8.3-intl \
  php8.3-mbstring \
  php8.3-soap \
  php8.3-xml \
  php8.3-xmlrpc \
  php8.3-zip \
  php8.3-redis \
  nfs-common \
  postgresql-client \
  redis-tools
```

**Bu paketlerin anlamı:**
- `nginx`: Web sunucusu
- `php8.3-*`: PHP ve Moodle'ın ihtiyaç duyduğu kütüphaneler
- `nfs-common`: NFS sunucusuna bağlanabilmek için
- `postgresql-client`: Veritabanına bağlanabilmek için
- `redis-tools`: Oturum yöneticisine bağlanabilmek için

---

### 2️⃣ NFS Bağlantısının Yapılandırılması

```bash
sudo mkdir -p /var/moodledata
sudo mount 192.168.1.50:/export/moodledata /var/moodledata
echo "192.168.1.50:/export/moodledata /var/moodledata nfs defaults,_netdev 0 0" | \
  sudo tee -a /etc/fstab

sudo chown -R www-data:www-data /var/moodledata
sudo chmod -R 755 /var/moodledata
ls -ld /var/moodledata
```

---

### 3️⃣ Moodle Kodlarının Hazırlanması

```bash
sudo mkdir -p /var/www/moodle
cd /tmp
git clone https://github.com/moodle/moodle.git --depth 1 --branch MOODLE_401_STABLE
sudo cp -r moodle/* /var/www/moodle/
sudo chown -R www-data:www-data /var/www/moodle
sudo chmod -R 755 /var/www/moodle
```

---

### 4️⃣ Moodle Konfigürasyon Dosyası (config.php)

```bash
sudo nano /var/www/moodle/config.php
```

```php
<?php
unset($CFG);
global $CFG;
$CFG = new stdClass();

// 1. VERITABANI AYARLARI
// ⚠️ PgBouncer üzerinden bağlanıyoruz (port 6432)
$CFG->dbtype    = 'pgsql';
$CFG->dblibrary = 'native';
$CFG->dbhost    = '192.168.1.100';   // PgBouncer IP
$CFG->dbport    = '6432';            // PgBouncer portu (PostgreSQL default 5432 değil!)
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = 'VERITABANI_SIFRENIZ';
$CFG->prefix    = 'mdl_';

// 2. WEB VE KLASÖR AYARLARI
$CFG->wwwroot   = 'https://moodle.argeyazilim.tr';
$CFG->dataroot  = '/var/moodledata';
$CFG->admin     = 'admin';
$CFG->directorypermissions = 0770;

// 3. REVERSE PROXY AYARLARI
$CFG->sslproxy = true;
$CFG->reverseproxy = true;

// 4. OTURUM YÖNETİMİ (Redis)
$CFG->session_handler_class = '\core\session\redis';
$CFG->session_redis_host = '192.168.1.185';
$CFG->session_redis_port = 6379;
$CFG->session_redis_auth = 'REDIS_SIFRENIZ';
$CFG->session_redis_prefix = 'moodle_session_';
$CFG->session_redis_acquire_lock_timeout = 120;
$CFG->session_redis_lock_expire = 7200;

// 5. İLETİŞİM AYARLARI
$CFG->smtphosts = 'smtp.example.com:25';
$CFG->noreplyaddress = 'noreply@moodle.argeyazilim.tr';
$CFG->supportemail = 'support@argeyazilim.tr';

// 6. GÜVENLİK AYARLARI
$CFG->disableupdatenotifications = true;

// 7. DEBUG (Üretimde kapalı)
$CFG->debug = DEBUG_NONE;
$CFG->debugdisplay = false;

require_once(__DIR__ . '/lib/setup.php');
```

---

### 5️⃣ Nginx Konfigürasyonu

```bash
sudo nano /etc/nginx/sites-available/moodle
```

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name moodle.argeyazilim.tr _;
    root /var/www/moodle;
    index index.php;
    client_max_body_size 100M;

    set_real_ip_from 192.168.1.10;
    real_ip_header X-Forwarded-For;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ [^/]\.php(/|$) {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_X_FORWARDED_FOR $proxy_add_x_forwarded_for;
        fastcgi_param HTTP_X_FORWARDED_PROTO $http_x_forwarded_proto;
        fastcgi_connect_timeout 60s;
        fastcgi_send_timeout 300s;
        fastcgi_read_timeout 300s;
        include fastcgi_params;
    }

    location ~ /\.ht { deny all; }
    location ~ /\.env { deny all; }
    location ~ ^/\.git { deny all; }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/moodle /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default 2>/dev/null || true
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl enable nginx
```

---

### 6️⃣ PHP-FPM Ayarlaması

```bash
sudo nano /etc/php/8.3/fpm/php.ini
```

```ini
upload_max_filesize = 100M
post_max_size = 100M
max_execution_time = 300
memory_limit = 256M
date.timezone = Europe/Istanbul
```

```bash
sudo systemctl restart php8.3-fpm
```

---

### 7️⃣ Cron Görevi (SADECE Node-1'de)

```bash
sudo crontab -u www-data -e
# Şu satırı ekle:
* * * * * /usr/bin/php /var/www/moodle/admin/cli/cron.php >/dev/null 2>&1
```

**Diğer 9 sunucuda cron görevi oluşturmayın!**

---

### 8️⃣ Moodle Veritabanını Başlatma

```bash
sudo -u www-data php /var/www/moodle/admin/cli/install_database.php \
  --lang=tr \
  --adminpass=GUCLU_YONETİCİ_SIFRENIZ \
  --agree-license
```

---

### 9️⃣ Yapılandırmayı Diğer Sunuculara Kopyalama

```bash
for i in {2..10}; do
  NODE_IP="192.168.1.$((10 + i))"
  echo "Node-$i'e kopyalanıyor ($NODE_IP)..."
  scp /var/www/moodle/config.php root@$NODE_IP:/var/www/moodle/
  scp /etc/nginx/sites-available/moodle root@$NODE_IP:/etc/nginx/sites-available/
  ssh root@$NODE_IP "systemctl restart nginx"
done
echo "Tüm sunucular güncellendi!"
```

---

## 📌 Bölüm 2: HAProxy ile Yük Dengeleme ve SSL Sonlandırma

### 1️⃣ HAProxy Kurulumu

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y haproxy
haproxy -v
```

---

### 2️⃣ SSL Sertifikasının Hazırlanması

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot certonly --standalone \
  -d moodle.argeyazilim.tr \
  --email your@email.com \
  --agree-tos

sudo mkdir -p /etc/haproxy/certs
sudo bash -c 'cat /etc/letsencrypt/live/moodle.argeyazilim.tr/fullchain.pem \
  /etc/letsencrypt/live/moodle.argeyazilim.tr/privkey.pem > \
  /etc/haproxy/certs/moodle.argeyazilim.tr.pem'
sudo chmod 600 /etc/haproxy/certs/moodle.argeyazilim.tr.pem
```

---

### 3️⃣ HAProxy Konfigürasyonu

```bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
sudo nano /etc/haproxy/haproxy.cfg
```

```haproxy
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    maxconn 4096
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

frontend http_front
    bind *:80
    mode http
    redirect scheme https code 301 if !{ ssl_fc }

frontend https_front
    bind *:443 ssl crt /etc/haproxy/certs/moodle.argeyazilim.tr.pem
    mode http
    option forwardfor
    http-request set-header X-Forwarded-Proto https
    http-request set-header X-Forwarded-Port 443
    default_backend moodle_cluster

backend moodle_cluster
    mode http
    balance leastconn
    option forwardfor
    http-request set-header X-Real-IP %[src]
    option httpchk GET /login/index.php HTTP/1.0
    http-check expect status 200
    default-server inter 2000 fall 3 rise 2
    server moodle-node-01 192.168.1.11:80 check
    server moodle-node-02 192.168.1.12:80 check
    server moodle-node-03 192.168.1.13:80 check
    server moodle-node-04 192.168.1.14:80 check
    server moodle-node-05 192.168.1.15:80 check
    server moodle-node-06 192.168.1.16:80 check
    server moodle-node-07 192.168.1.17:80 check
    server moodle-node-08 192.168.1.18:80 check
    server moodle-node-09 192.168.1.19:80 check
    server moodle-node-10 192.168.1.20:80 check

listen stats
    bind *:8404
    stats enable
    stats uri /haproxy_stats
    stats refresh 10s
    stats auth admin:GUCLU_BIR_SIFRE_YAZIN
    mode http
```

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
sudo systemctl enable haproxy
```

---

## 📌 Bölüm 3: PostgreSQL Kurulumu ve PgBouncer ile SQL Yük Dengeleme

Bu bölümde PostgreSQL veritabanı sunucusunu sıfırdan kuracağız. Ardından **PgBouncer** bağlantı havuzlayıcısını kurarak 10 Moodle sunucusunun yüzlerce eşzamanlı bağlantısını verimli şekilde yöneteceğiz.

**Neden PgBouncer?**
PostgreSQL her bağlantı için ayrı bir süreç oluşturur; bu maliyetlidir. 10 Moodle sunucusu × her sunucuda çok sayıda PHP worker = binlerce eş zamanlı bağlantı. PgBouncer bu bağlantıları havuzlayarak PostgreSQL'e çok daha az bağlantı gönderir.

```
Moodle Sunucuları (10x)          PgBouncer          PostgreSQL
  Node-1  ──┐                  ┌──────────┐        ┌──────────┐
  Node-2  ──┤ ~1000 bağlantı  │          │ ~50    │          │
  ...     ──┼────────────────▶️│ Havuz    │───────▶️│ Primary  │
  Node-10 ──┘                  │ :6432    │        │ :5432    │
                                └──────────┘        └──────────┘
```

---

### 1️⃣ PostgreSQL Kurulumu (Primary Sunucu: 192.168.1.100)

#### 1.1 PostgreSQL'i Kur

```bash
# Sistem paketlerini güncelle
sudo apt update && sudo apt upgrade -y

# PostgreSQL resmi deposunu ekle (güncel sürüm için)
sudo apt install -y curl ca-certificates gnupg
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | \
  sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg

echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | \
  sudo tee /etc/apt/sources.list.d/pgdg.list

sudo apt update

# PostgreSQL 16'yı kur
sudo apt install -y postgresql-16 postgresql-client-16 postgresql-contrib-16

# Servis durumunu kontrol et
sudo systemctl status postgresql
# "active (running)" görmelisiniz

# Sürümü kontrol et
psql --version
```

#### 1.2 PostgreSQL'i Başlangıçta Başlat

```bash
sudo systemctl enable postgresql
sudo systemctl start postgresql
```

---

### 2️⃣ Veritabanı ve Kullanıcı Oluşturma

```bash
# PostgreSQL yönetici hesabına geç
sudo -u postgres psql
```

PostgreSQL komut isteminde aşağıdaki SQL komutlarını çalıştır:

```sql
-- Moodle için güçlü bir şifre ile kullanıcı oluştur
-- ⚠️ 'VERITABANI_SIFRENIZ' kısmını güçlü bir şifreyle değiştirin
CREATE USER moodleuser WITH PASSWORD 'VERITABANI_SIFRENIZ';

-- Moodle veritabanını oluştur
-- UTF8 karakter seti Moodle için zorunludur
CREATE DATABASE moodle
    WITH OWNER = moodleuser
    ENCODING = 'UTF8'
    LC_COLLATE = 'tr_TR.UTF-8'
    LC_CTYPE = 'tr_TR.UTF-8'
    TEMPLATE = template0;

-- Kullanıcıya veritabanı üzerinde tam yetki ver
GRANT ALL PRIVILEGES ON DATABASE moodle TO moodleuser;

-- Scalelite için ayrı veritabanı ve kullanıcı (Bölüm 4 için)
CREATE USER scalelite WITH PASSWORD 'SCALELITE_SIFRENIZ';
CREATE DATABASE scalelite_production
    WITH OWNER = scalelite
    ENCODING = 'UTF8'
    LC_COLLATE = 'tr_TR.UTF-8'
    LC_CTYPE = 'tr_TR.UTF-8'
    TEMPLATE = template0;
GRANT ALL PRIVILEGES ON DATABASE scalelite_production TO scalelite;

-- Oluşturulanları listele
\l

-- Çıkış
\q
```

**Türkçe locale yoksa** (`tr_TR.UTF-8` hatası alırsanız):

```bash
# PostgreSQL dışında, sistem terminalinde:
sudo locale-gen tr_TR.UTF-8
sudo update-locale

# PostgreSQL'i yeniden başlat
sudo systemctl restart postgresql

# Tekrar dene
sudo -u postgres psql
```

---

### 3️⃣ PostgreSQL Ağ Erişim Ayarları

PostgreSQL varsayılan olarak sadece `localhost`'tan bağlantı kabul eder. Moodle sunucularının bağlanabilmesi için bu ayarı değiştirmeliyiz.

#### 3.1 Dinleme Adresini Ayarla

```bash
sudo nano /etc/postgresql/16/main/postgresql.conf
```

Şu satırı bulup değiştir:

```ini
# Eski:
#listen_addresses = 'localhost'

# Yeni (tüm arayüzlerde dinle):
listen_addresses = '*'
```

Aynı dosyada performans ayarlarını da düzenle:

```ini
# Bağlantı limiti (PgBouncer ile 100 yeterli)
max_connections = 200

# Bellek ayarları (sunucu RAM'inin %25'i önerilir)
# 8GB RAM için:
shared_buffers = 2GB
effective_cache_size = 6GB
work_mem = 64MB
maintenance_work_mem = 512MB

# WAL (Write-Ahead Log) ayarları
wal_level = replica          # Replikasyon için (opsiyonel)
max_wal_senders = 3          # Replikasyon bağlantıları
wal_keep_size = 1GB

# Zaman dilimi
timezone = 'Europe/Istanbul'
log_timezone = 'Europe/Istanbul'
```

#### 3.2 İstemci Kimlik Doğrulama (pg_hba.conf)

```bash
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

Dosyanın sonuna şu satırları ekle:

```
# ============================================================
# Moodle Altyapısı Erişim Kuralları
# ============================================================
# Format: TYPE  DATABASE    USER        ADDRESS             METHOD

# PgBouncer'ın localhost'tan bağlanmasına izin ver
host    moodle          moodleuser  127.0.0.1/32        scram-sha-256
host    moodle          moodleuser  ::1/128             scram-sha-256

# Scalelite'ın bağlanmasına izin ver
host    scalelite_production  scalelite  127.0.0.1/32    scram-sha-256

# ⚠️ Alternatif: Tüm Moodle sunucularından direkt erişim (PgBouncer olmadan)
# Eğer PgBouncer kullanmıyorsanız bu satırları açın:
# host    moodle    moodleuser    192.168.1.11/32    scram-sha-256
# host    moodle    moodleuser    192.168.1.12/32    scram-sha-256
# ... 192.168.1.20'ye kadar

# ⚠️ Eğer PgBouncer ayrı sunucudaysa, PgBouncer sunucu IP'sini ekle:
# host    moodle    moodleuser    192.168.1.101/32    scram-sha-256
```

#### 3.3 PostgreSQL'i Yeniden Başlat

```bash
sudo systemctl restart postgresql

# Servis durumunu kontrol et
sudo systemctl status postgresql

# PostgreSQL'in 5432 portunu dinleyip dinlemediğini kontrol et
ss -tlnp | grep 5432
# Çıktı: LISTEN 0 128 0.0.0.0:5432 şeklinde olmalı
```

---

### 4️⃣ Bağlantıyı Test Et

Moodle sunucularından herhangi birinde (Ör: Node-1):

```bash
# PostgreSQL istemcisi ile bağlantı testi
psql -h 192.168.1.100 -p 5432 -U moodleuser -d moodle

# Bağlantı başarılı olursa şu çıktıyı görürsünüz:
# psql (16.x)
# SSL connection (...)
# moodle=>

# Bir test sorgusu çalıştır
moodle=> SELECT version();
moodle=> \q
```

---

### 5️⃣ PgBouncer Kurulumu (SQL Bağlantı Havuzlayıcı)

PgBouncer, PostgreSQL ile aynı sunucuya (192.168.1.100) kurulacak ve port 6432'de dinleyecektir.

#### 5.1 PgBouncer'ı Kur

```bash
# PgBouncer'ı kur
sudo apt install -y pgbouncer

# Sürümü kontrol et
pgbouncer --version
```

#### 5.2 PgBouncer Veritabanı Tanımları

```bash
sudo nano /etc/pgbouncer/pgbouncer.ini
```

**Tüm içeriği sil ve şunları yapıştır:**

```ini
; ============================================================================
; PgBouncer Konfigürasyonu - Moodle HA Altyapısı
; ============================================================================

[databases]
; Moodle veritabanı - localhost'taki PostgreSQL'e yönlendir
moodle = host=127.0.0.1 port=5432 dbname=moodle

; Scalelite veritabanı
scalelite_production = host=127.0.0.1 port=5432 dbname=scalelite_production

[pgbouncer]
; ============================================================================
; Bağlantı Ayarları
; ============================================================================
; Hangi adres ve portta dinleyeceği
listen_addr = 0.0.0.0
listen_port = 6432

; ============================================================================
; Havuzlama Modu
; ============================================================================
; transaction: Her SQL transaction için bağlantı ver (Moodle için önerilen)
; session: Kullanıcı bağlandığında bağlantı ver (daha basit ama daha az verimli)
; statement: Her SQL cümlesi için bağlantı ver (en verimli ama kısıtlı)
pool_mode = transaction

; ============================================================================
; Havuz Boyutları
; ============================================================================
; PostgreSQL'e açılacak maksimum bağlantı sayısı (veritabanı başına)
max_client_conn = 1000     ; Moodle sunucularından gelen toplam bağlantı
default_pool_size = 50     ; PostgreSQL'e gidecek gerçek bağlantı sayısı
min_pool_size = 10         ; Hazırda tutulacak minimum bağlantı
reserve_pool_size = 5      ; Yoğunluk durumunda ekstra bağlantı
reserve_pool_timeout = 5   ; Ekstra bağlantı için bekleme süresi (saniye)

; ============================================================================
; Zaman Aşımı Ayarları
; ============================================================================
server_idle_timeout = 600      ; Boş bağlantı kapatma süresi (saniye)
client_idle_timeout = 0        ; İstemci bağlantı zaman aşımı (0=kapalı)
query_timeout = 0              ; Sorgu zaman aşımı (0=kapalı)
client_login_timeout = 60      ; Giriş zaman aşımı (saniye)

; ============================================================================
; Kimlik Doğrulama
; ============================================================================
; Kullanıcı şifrelerinin saklandığı dosya
auth_file = /etc/pgbouncer/userlist.txt
; Kimlik doğrulama türü
auth_type = scram-sha-256

; ============================================================================
; Yönetim
; ============================================================================
; PgBouncer yönetim konsoluna erişecek kullanıcılar (virgülle ayır)
admin_users = postgres
; İstatistikleri görebilecek kullanıcılar
stats_users = postgres, moodleuser

; ============================================================================
; Loglama
; ============================================================================
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
log_connections = 0    ; Bağlantıları logla (0=kapalı, üretimde kapanmalı)
log_disconnections = 0
log_stats = 1          ; İstatistikleri logla (600 saniyede bir)
stats_period = 600

; ============================================================================
; Sunucu TLS (PostgreSQL'e şifreli bağlantı - isteğe bağlı)
; ============================================================================
; server_tls_sslmode = prefer
```

#### 5.3 Kullanıcı Şifre Dosyası

PgBouncer, PostgreSQL kullanıcılarının şifrelerini kendi dosyasında saklar:

```bash
# Moodle kullanıcısı için şifreyi PostgreSQL'den al (SCRAM formatında)
sudo -u postgres psql -c "SELECT usename, passwd FROM pg_shadow WHERE usename='moodleuser';"
```

Çıktıdaki şifreyi kopyala ve userlist dosyasına ekle:

```bash
sudo nano /etc/pgbouncer/userlist.txt
```

```
"moodleuser" "SCRAM-SHA-256$4096:...buraya_pg_shadow_ciktisini_yapistirin..."
"scalelite" "SCRAM-SHA-256$4096:...buraya_scalelite_sifresini_yapistirin..."
```

**Alternatif — düz metin şifre (geliştirme ortamı için):**

```bash
# Eğer pg_shadow çıktısını kullanmak istemiyorsanız:
# auth_type = md5 olarak değiştirip aşağıdaki formatı kullanabilirsiniz:
"moodleuser" "VERITABANI_SIFRENIZ"
```

#### 5.4 PgBouncer'ı Başlat

```bash
# Dosya izinlerini ayarla
sudo chown postgres:postgres /etc/pgbouncer/userlist.txt
sudo chmod 600 /etc/pgbouncer/userlist.txt

# Konfigürasyonu doğrula
sudo -u postgres pgbouncer -d /etc/pgbouncer/pgbouncer.ini

# Hata yoksa servisi başlat
sudo systemctl restart pgbouncer
sudo systemctl enable pgbouncer
sudo systemctl status pgbouncer

# PgBouncer'ın 6432 portunu dinleyip dinlemediğini kontrol et
ss -tlnp | grep 6432
# Çıktı: LISTEN 0 128 0.0.0.0:6432 şeklinde olmalı
```

---

### 6️⃣ PgBouncer Bağlantısını Test Et

**Node-1 üzerinde (Moodle sunucusundan):**

```bash
# PgBouncer üzerinden veritabanına bağlan
psql -h 192.168.1.100 -p 6432 -U moodleuser -d moodle

# Bağlantı başarılıysa:
moodle=> SELECT current_database(), inet_server_addr(), inet_server_port();
# current_database: moodle
# inet_server_addr: 127.0.0.1
# inet_server_port: 5432
moodle=> \q
```

---

### 7️⃣ PgBouncer İstatistik Konsolu

PgBouncer'ın yerleşik yönetim konsoluna bağlanabilirsiniz:

```bash
# PgBouncer yönetim konsoluna bağlan
psql -h 127.0.0.1 -p 6432 -U postgres pgbouncer
```

Yararlı komutlar:

```sql
-- Aktif havuz durumunu gör
SHOW POOLS;

-- Bağlantı istatistiklerini gör
SHOW STATS;

-- Veritabanı bağlantılarını gör
SHOW DATABASES;

-- Aktif istemcileri gör
SHOW CLIENTS;

-- Aktif sunucu bağlantılarını gör
SHOW SERVERS;

-- Konfigürasyonu yeniden yükle (restart gerektirmez)
RELOAD;

-- Çıkış
QUIT;
```

Örnek `SHOW POOLS` çıktısı:

```
 database  |   user    | cl_active | cl_waiting | sv_active | sv_idle | sv_used | maxwait
-----------+-----------+-----------+------------+-----------+---------+---------+--------
 moodle    | moodleuser|        45 |          0 |        45 |       5 |       0 |       0
```

- `cl_active`: PgBouncer'a bağlı aktif Moodle istemcileri
- `sv_active`: PostgreSQL'e açık aktif bağlantılar
- `maxwait`: Bağlantı bekleyen istemci sayısı (0 olmalı, yüksekse pool_size artır)

---

### 8️⃣ PostgreSQL Güvenlik Duvarı Ayarları

```bash
# UFW (Ubuntu güvenlik duvarı) aktifse:
sudo ufw allow from 192.168.1.0/24 to any port 5432   # PostgreSQL (yalnızca iç ağ)
sudo ufw allow from 192.168.1.0/24 to any port 6432   # PgBouncer (yalnızca iç ağ)

# Dışarıdan PostgreSQL/PgBouncer erişimini ENGELLE:
sudo ufw deny 5432
sudo ufw deny 6432

# Durumu kontrol et
sudo ufw status
```

---

### 9️⃣ (Opsiyonel) PostgreSQL Streaming Replikasyonu

Yüksek erişilebilirlik için bir Replica sunucu (192.168.1.101) ekleyebilirsiniz. Bu sunucu okuma sorgularını üstlenir ve Primary çökerse devreye girer.

#### Primary Sunucuda (192.168.1.100):

```bash
# Replikasyon kullanıcısı oluştur
sudo -u postgres psql
```

```sql
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'REPLIKASYON_SIFRESI';
\q
```

```bash
# pg_hba.conf'a replikasyon erişimi ekle
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

Şunu ekle:

```
# Replica sunucudan replikasyon bağlantısı
host    replication     replicator    192.168.1.101/32    scram-sha-256
```

```bash
sudo systemctl reload postgresql
```

#### Replica Sunucuda (192.168.1.101):

```bash
# PostgreSQL'i aynı şekilde kur (adım 1)

# Mevcut veriyi Primary'den kopyala
sudo systemctl stop postgresql
sudo rm -rf /var/lib/postgresql/16/main/*

sudo -u postgres pg_basebackup \
  -h 192.168.1.100 \
  -U replicator \
  -D /var/lib/postgresql/16/main \
  -P \
  -Xs \
  -R

# -R bayrağı otomatik olarak standby.signal ve postgresql.auto.conf oluşturur
sudo systemctl start postgresql

# Replica çalışıyor mu kontrol et
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
# Çıktı: t (true = replica modunda)
```

#### PgBouncer'a Read Replica Ekle (Opsiyonel):

```bash
sudo nano /etc/pgbouncer/pgbouncer.ini
```

`[databases]` bölümüne ekle:

```ini
; Okuma sorguları için replica (Moodle bunu otomatik kullanmaz;
; ilerleyen sürümler veya özel eklentilerle yönlendirilebilir)
moodle_read = host=192.168.1.101 port=5432 dbname=moodle
```

```bash
sudo systemctl reload pgbouncer
```

---

### 🔁 Bölüm 3 Özet: Mimari

```
10x Moodle Sunucusu
  (192.168.1.11-20)
        │
        │  port 6432 (PgBouncer)
        ▼
┌───────────────────┐
│   PgBouncer       │  max_client_conn=1000
│   192.168.1.100   │  default_pool_size=50
│   :6432           │  pool_mode=transaction
└───────────────────┘
        │
        │  port 5432 (PostgreSQL)
        ▼
┌───────────────────┐      ┌───────────────────┐
│   PostgreSQL      │─────▶️│   PostgreSQL      │
│   Primary         │      │   Replica         │
│   192.168.1.100   │      │   192.168.1.101   │
│   :5432 (R/W)     │      │   :5432 (R only)  │
└───────────────────┘      └───────────────────┘
```

---

## 📌 Bölüm 4: Scalelite ve BigBlueButton Kümesi

### 1️⃣ Scalelite İçin NFS Ayarı

```bash
sudo mkdir -p /var/bigbluebutton/spool
sudo mkdir -p /var/bigbluebutton/published
sudo mkdir -p /var/bigbluebutton/unpublished

sudo mount 192.168.1.50:/export/bbb_spool /var/bigbluebutton/spool
sudo mount 192.168.1.50:/export/bbb_published /var/bigbluebutton/published
sudo mount 192.168.1.50:/export/bbb_unpublished /var/bigbluebutton/unpublished

echo "192.168.1.50:/export/bbb_spool /var/bigbluebutton/spool nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
echo "192.168.1.50:/export/bbb_published /var/bigbluebutton/published nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
echo "192.168.1.50:/export/bbb_unpublished /var/bigbluebutton/unpublished nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab

sudo chmod -R 755 /var/bigbluebutton/
```

---

### 2️⃣ Scalelite Ortam Değişkenleri (.env)

```bash
sudo nano /var/www/scalelite/.env
```

```bash
URL_HOST=scalelite.argeyazilim.tr
SCALELITE_TAG=scalelite.argeyazilim.tr

# Komut: openssl rand -hex 32
SECRET_KEY_BASE=BURAYA_OLUSTURULAN_64_KARAKTERLIK_ANAHTAR
LOADBALANCER_SECRET=BURAYA_MOODLE_ICIN_GUCLU_SIFRE

# ⚠️ PgBouncer üzerinden bağlanıyoruz (port 6432)
DATABASE_URL=postgres://scalelite:SIFRE@192.168.1.100:6432/scalelite_production

REDIS_URL=redis://192.168.1.185:6379/0
RAILS_ENV=production
```

---

### 3️⃣ Scalelite Poller Servisi

```bash
sudo nano /etc/systemd/system/scalelite-poller.service
```

```ini
[Unit]
Description=Scalelite Poller - BigBlueButton Sağlık Kontrolü
After=network.target

[Service]
Type=simple
User=scalelite
WorkingDirectory=/var/www/scalelite
EnvironmentFile=/var/www/scalelite/.env
ExecStart=/bin/bash -c 'while true; do RAILS_ENV=production bundle exec rake poll:all; sleep 60; done'
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable scalelite-poller
sudo systemctl start scalelite-poller
```

---

### 4️⃣ BigBlueButton Sunucularını Scalelite'a Ekleme

```bash
sudo bbb-conf --secret
# Çıktıdaki URL ve Secret değerlerini not al

cd /var/www/scalelite
RAILS_ENV=production bundle exec rake servers:add[\
https://bbb-node-01.argeyazilim.tr/bigbluebutton/api,\
SUNUCU_1_SECRET_KODU\
]

RAILS_ENV=production bundle exec rake servers
RAILS_ENV=production bundle exec rake servers:enable[1]
```

---

### 5️⃣ Moodle'da Scalelite Konfigürasyonu

**Site Yönetimi → Eklentiler → BigBlueButton → Ayarlar:**

| Ayar | Değer |
|------|-------|
| **BigBlueButton Server URL** | `https://scalelite.argeyazilim.tr/bigbluebutton/api` |
| **Shared Secret** | `.env` dosyasındaki `LOADBALANCER_SECRET` |

---

## 🔧 Sorun Giderme İpuçları

### 🟥 Problem 1: Moodle Sunucuları HAProxy'de DOWN Gösteriliyor

```bash
ping 192.168.1.11
ssh user@192.168.1.11 "systemctl status nginx"
curl http://192.168.1.11/login/index.php
sudo tail -f /var/log/haproxy.log
```

### 🟥 Problem 2: Kullanıcılar Oturumdan Düşüyor

```bash
redis-cli -h 192.168.1.185
> ping
# Çıktı: PONG

grep -A 5 "session_redis" /var/www/moodle/config.php
```

### 🟥 Problem 3: Dosya Yükleme Başarısız Oluyor

```bash
df -h /var/moodledata
ls -ld /var/moodledata
touch /var/moodledata/test.txt && rm /var/moodledata/test.txt
```

### 🟥 Problem 4: SSL Sertifikası Hatası

```bash
openssl x509 -in /etc/haproxy/certs/moodle.argeyazilim.tr.pem -noout -dates
sudo certbot renew --dry-run
sudo systemctl restart haproxy
```

### 🟥 Problem 5: PostgreSQL Bağlantı Hatası

```bash
# PostgreSQL çalışıyor mu?
sudo systemctl status postgresql

# PostgreSQL loglarını kontrol et
sudo tail -50 /var/log/postgresql/postgresql-16-main.log

# Uzaktan bağlantı testi
psql -h 192.168.1.100 -p 5432 -U moodleuser -d moodle -c "SELECT 1;"

# pg_hba.conf ayarlarını kontrol et
sudo grep -v "^#" /etc/postgresql/16/main/pg_hba.conf | grep -v "^$"
```

### 🟥 Problem 6: PgBouncer Bağlantı Hatası

```bash
# PgBouncer çalışıyor mu?
sudo systemctl status pgbouncer

# PgBouncer loglarını kontrol et
sudo tail -50 /var/log/postgresql/pgbouncer.log

# PgBouncer portunu kontrol et
ss -tlnp | grep 6432

# Bağlantı testi
psql -h 192.168.1.100 -p 6432 -U moodleuser -d moodle -c "SELECT 1;"

# Havuz durumunu kontrol et
psql -h 127.0.0.1 -p 6432 -U postgres pgbouncer -c "SHOW POOLS;"

# Konfigürasyonu yeniden yükle (restart gerektirmez)
psql -h 127.0.0.1 -p 6432 -U postgres pgbouncer -c "RELOAD;"
```

### 🟥 Problem 7: PgBouncer "prepared statement" Hatası

Moodle bazı sürümlerde prepared statement kullanır. Transaction pool modu bununla uyumsuz olabilir:

```bash
sudo nano /etc/pgbouncer/pgbouncer.ini
```

```ini
; transaction yerine session moduna geç (daha az verimli ama uyumlu)
pool_mode = session

; Veya prepared statement'ları devre dışı bırak (PHP PDO için):
; Moodle config.php'ye şunu ekle:
; $CFG->dboptions = ['options' => '--disable_prepared_transactions=1'];
```

```bash
sudo systemctl restart pgbouncer
```

---

## 📚 Ek Kaynaklar

- [Moodle Resmi Dokümantasyonu](https://docs.moodle.org)
- [Moodle Küme Kurulumu](https://docs.moodle.org/en/Setting_up_a_cluster)
- [HAProxy Dokümantasyonu](http://www.haproxy.org/)
- [PostgreSQL Dokümantasyonu](https://www.postgresql.org/docs/)
- [PgBouncer Dokümantasyonu](https://www.pgbouncer.org/config.html)
- [BigBlueButton Dokümantasyonu](https://docs.bigbluebutton.org/)
- [Scalelite GitHub](https://github.com/blindsidenetworks/scalelite)
- [Redis Dokümantasyonu](https://redis.io/documentation)

---

**Son Güncelleme:** Haziran 2026  
**Versiyon:** 3.0 (PostgreSQL + PgBouncer Eklendi)
