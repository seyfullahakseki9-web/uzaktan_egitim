# 🎓 Yüksek Erişilebilir (HA) Uzaktan Eğitim Altyapısı

Binlerce öğrenciye aynı anda hizmet verebilmek için tasarlanmış, **Yüksek Erişilebilirlik (HA)** ve **Ölçeklenebilirlik** prensiplerine dayalı kapsamlı bir Moodle altyapısının kurulumu rehberidir.

---

## 📋 İçindekiler

1. [Sistem Mimarisi](https://www.google.com/search?q=%23sistem-mimarisi)
2. [Bölüm 1: Moodle Uygulama Sunucularının Kurulumu](https://www.google.com/search?q=%23b%C3%B6l%C3%BCm-1-moodle-uygulama-sunucular%C4%B1n%C4%B1n-kurulumu)
3. [Bölüm 2: HAProxy ile Yük Dengeleme](https://www.google.com/search?q=%23b%C3%B6l%C3%BCm-2-haproxy-ile-y%C3%BCk-dengeleme)
4. [Bölüm 3: PostgreSQL Kurulumu ve PgBouncer ile SQL Yük Dengeleme](https://www.google.com/search?q=%23b%C3%B6l%C3%BCm-3-postgresql-kurulumu-ve-pgbouncer-ile-sql-y%C3%BCk-dengeleme)
5. [Bölüm 4: Ortak Depolama Katmanı (NFS Server ve İzin Yönetimi)](https://www.google.com/search?q=%23b%C3%B6l%C3%BCm-4-ortak-depolama-katman%C4%B1-nfs-server-ve-izin-y%C3%B6netimi)
6. [Bölüm 5: Scalelite ve BigBlueButton Kümesi](https://www.google.com/search?q=%23b%C3%B6l%C3%BCm-5-scalelite-ve-bigbluebutton-k%C3%BCmesi)
7. [Sorun Giderme İpuçları](https://www.google.com/search?q=%23sorun-giderme-ipu%C3%A7lar%C4%B1)

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
  │Nginx │    │Nginx │    │Nginx │     (192.168.1.11-20)
  │PHP   │    │PHP   │    │PHP   │
  └──────┘    └──────┘    └──────┘
     │            │            │
     └────────────┼────────────┘
                  │
        ┌─────────┼──────────┐
        ▼         ▼          ▼
    ┌──────────┐ ┌────────┐ ┌────────────────────────┐
    │PgBouncer │ │ Redis  │ │    NFS Depolama        │
    │(SQL Bal.)│ │Session │ │ (MoodleData & BBB)    │
    │192.168.1.│ │192.168.│ │ IP: 192.168.1.87       │
    │101 :6432 │ │1.185   │ │ UID 33 / UID 2000      │
    └──────────┘ └────────┘ └────────────────────────┘
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
│ Canlı Dersler (BigBlueButton Kümesi)    │
├─────────────────────────────────────────┤
│  Scalelite Yük Dengeleyici              │
│  (192.168.1.60)                         │
│         │                               │
│    ┌────┴────┬─────────┐                │
│    ▼         ▼         ▼                │
│  BBB-1    BBB-2    BBB-N                │
└─────────────────────────────────────────┘

```

### 🖥️ Bileşen Özeti

| Bileşen | Sayı | Görev | IP Örneği |
| --- | --- | --- | --- |
| **HAProxy** | 1 | SSL Sonlandırması, Trafik Dağıtımı | 192.168.1.10 |
| **Moodle Node** | 10 | Eğitim Platformu (Nginx + PHP) | 192.168.1.11-20 |
| **PostgreSQL Primary** | 1 | Ana Veritabanı (Yazma) | 192.168.1.100 |
| **PostgreSQL Replica** | 1 | Okuma Replikası (Opsiyonel) | 192.168.1.101 |
| **PgBouncer** | 1 | SQL Bağlantı Havuzu / Yük Dengeleme | 192.168.1.100 :6432 |
| **Redis** | 1 | Oturum ve Önbellek | 192.168.1.185 |
| **NFS Sunucusu** | 1 | Ortak Dosya Depolama (MoodleData & Kayıtlar) | 192.168.1.87 |
| **Scalelite** | 1 | BBB Yük Dengeleyici | 192.168.1.60 |
| **BigBlueButton** | N | Canlı Ders Sunucuları | 192.168.1.179+ |

---

## 📌 Bölüm 1: Moodle Uygulama Sunucularının Kurulumu

Bu bölümde, arkasında HAProxy'nin olacağı 10 adet Moodle sunucusundan **ilkini (Node-1)** kuracağız. Ardından konfigürasyon diğer sunuculara kopyalanabilir.

### ⚠️ Ön Koşullar

* Ubuntu 20.04+ veya Debian sunucusu
* Root veya sudo yetkisi
* Ağda erişilebilir PostgreSQL sunucusu
* Ağda erişilebilir Redis sunucusu
* Ağda erişilebilir NFS sunucusu

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

---

### 2️⃣ Moodle Kodlarının Hazırlanması

```bash
sudo mkdir -p /var/www/moodle
cd /tmp
git clone https://github.com/moodle/moodle.git --depth 1 --branch MOODLE_401_STABLE
sudo cp -r moodle/* /var/www/moodle/
sudo chown -R www-data:www-data /var/www/moodle
sudo chmod -R 755 /var/www/moodle

```

---

### 3️⃣ Moodle Konfigürasyon Dosyası (config.php)

```bash
sudo nano /var/www/moodle/config.php

```

```php
<?php
unset($CFG);
global $CFG;
$CFG = new stdClass();

// 1. VERITABANI AYARLARI
$CFG->dbtype    = 'pgsql';
$CFG->dblibrary = 'native';
$CFG->dbhost    = '192.168.1.100';   // PgBouncer IP
$CFG->dbport    = '6432';            // PgBouncer portu
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = 'VERITABANI_SIFRENIZ';
$CFG->prefix    = 'mdl_';

// 2. WEB VE KLASÖR AYARLARI
$CFG->wwwroot   = 'https://moodle.argeyazilim.tr';
$CFG->dataroot  = '/var/moodledata'; // NFS Paylaşımı
$CFG->localcachedir = '/var/moodle_localcache'; // ⚠️ KRİTİK: NFS üzerinde tutulmamalı, yerel diskte olmalı!
$CFG->admin     = 'admin';
$CFG->directorypermissions = 0775;

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

require_once(__DIR__ . '/lib/setup.php');

```

---

### 4️⃣ Nginx Konfigürasyonu

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
        include fastcgi_params;
    }
}

```

```bash
sudo ln -s /etc/nginx/sites-available/moodle /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default 2>/dev/null || true
sudo systemctl restart nginx

```

---

### 5️⃣ Cron Görevi (SADECE Node-1'de)

```bash
sudo crontab -u www-data -e
# Şu satırı ekle:
* * * * * /usr/bin/php /var/www/moodle/admin/cli/cron.php >/dev/null 2>&1

```

---

## 📌 Bölüm 2: HAProxy ile Yük Dengeleme ve SSL Sonlandırma

### 1️⃣ HAProxy Kurulumu ve Sertifika

```bash
sudo apt update && sudo apt install -y haproxy certbot python3-certbot-nginx

sudo certbot certonly --standalone -d moodle.argeyazilim.tr --email info@argeyazilim.tr --agree-tos

sudo mkdir -p /etc/haproxy/certs
sudo bash -c 'cat /etc/letsencrypt/live/moodle.argeyazilim.tr/fullchain.pem \
  /etc/letsencrypt/live/moodle.argeyazilim.tr/privkey.pem > \
  /etc/haproxy/certs/moodle.argeyazilim.tr.pem'
sudo chmod 600 /etc/haproxy/certs/moodle.argeyazilim.tr.pem

```

---

### 2️⃣ HAProxy Konfigürasyonu

```bash
sudo nano /etc/haproxy/haproxy.cfg

```

```haproxy
frontend http_front
    bind *:80
    mode http
    redirect scheme https code 301 if !{ ssl_fc }

frontend https_front
    bind *:443 ssl crt /etc/haproxy/certs/moodle.argeyazilim.tr.pem
    mode http
    option forwardfor
    http-request set-header X-Forwarded-Proto https
    default_backend moodle_cluster

backend moodle_cluster
    mode http
    balance leastconn
    option httpchk GET /login/index.php HTTP/1.0
    http-check expect status 200
    server moodle-node-01 192.168.1.11:80 check
    server moodle-node-02 192.168.1.12:80 check

```

---

## 📌 Bölüm 3: PostgreSQL Kurulumu ve PgBouncer ile SQL Yük Dengeleme

```bash
sudo apt install -y postgresql-16 pgbouncer

```

### 1️⃣ PgBouncer Yapılandırması (`/etc/pgbouncer/pgbouncer.ini`)

```ini
[databases]
moodle = host=127.0.0.1 port=5432 dbname=moodle
scalelite_production = host=127.0.0.1 port=5432 dbname=scalelite_production

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 50
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

```

---

## 📌 Bölüm 4: Ortak Depolama Katmanı (NFS Server ve İzin Yönetimi)

Dağıtık mimarinin sağlıklı çalışabilmesi için tüm Moodle web cluster düğümlerinin, BigBlueButton sunucularının ve Scalelite yük dengeleyicisinin ortak bir depolama havuzuna kararlı ve doğru yetkilerle erişmesi şarttır.

---

### 1️⃣ NFS Sunucusunun Kurulumu (Sunucu IP: 192.168.1.87)

Ana NFS makinesinde gerekli servis paketlerini kurup paylaşım dizinlerini oluşturalım:

```bash
# Paket kurulumu
sudo apt update && sudo apt install -y nfs-kernel-server

# MoodleData ve Scalelite/BBB için ortak dizinlerin oluşturulması
sudo mkdir -p /mnt/moodledata
sudo mkdir -p /mnt/scalelite-recordings/var/bigbluebutton/spool
sudo mkdir -p /mnt/scalelite-recordings/var/bigbluebutton/published
sudo mkdir -p /mnt/scalelite-recordings/var/bigbluebutton/unpublished

```

---

### 2️⃣ ⚠️ En Kritik Adım: Kullanıcı Kimlik ve İzin Eşleştirmesi (UID/GID)

Dağıtık mimarilerde yaşanan inatçı `Permission denied` hatalarının ana kaynağı sunucular ile konteynerlerin içindeki Linux kullanıcı ID'lerinin uyuşmamasıdır.

* **Moodle Sunucuları İçin (`www-data`):** Ubuntu/Debian sistemlerde Apache/Nginx web sunucusu varsayılan olarak **UID: 33** kullanır.
* **Scalelite Konteynerleri İçin (`scalelite`):** Docker üzerinde çalışan Scalelite mikroservisleri (API, Importer, Poller) güvenlik gereği root olarak değil, **UID: 2000** kimliğine sahip `scalelite` kullanıcısı ile çalışır.

Bu nedenle NFS sunucusu üzerindeki dosya izinleri bu kimliklere göre mühürlenmelidir:

```bash
# Moodle verileri için sahibini www-data (33) yapalım
sudo chown -R 33:33 /mnt/moodledata
sudo chmod -R 775 /mnt/moodledata

# Scalelite/BBB kayıt süreçleri için yetkiyi doğrudan UID 2000'e (Scalelite) devredelim
sudo chown -R 2000:2000 /mnt/scalelite-recordings/
sudo chmod -R 777 /mnt/scalelite-recordings/

```

---

### 3️⃣ Dışa Aktarma Kuralları (`/etc/exports`)

NFS sunucusunun iç ağdaki istemcilere (`192.168.1.0/24`) güvenli ve sınırlandırılmamış erişim sunması için exports ayarlarını yapılandıralım:

```bash
sudo nano /etc/exports

```

Aşağıdaki satırları dosyanın sonuna ekleyin:

```text
/mnt/moodledata 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash,insecure)
/mnt/scalelite-recordings/var/bigbluebutton/spool 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash,insecure)
/mnt/scalelite-recordings/var/bigbluebutton/published 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash,insecure)
/mnt/scalelite-recordings/var/bigbluebutton/unpublished 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash,insecure)

```

> 💡 `no_root_squash`: İstemci makinelerdeki root isteklerinin NFS üzerinde yetkisiz bir kullanıcıya (`nobody`) dönüştürülmesini engeller ve altyapısal izin yönetimini korur.

Ayarları canlıya alalım:

```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server

```

---

### 4️⃣ İstemci (Client) Makinelerin Yapılandırılması

#### A. Moodle Web Sunucularında Bağlantı (Node 1-10)

Web sunucularında ortak `moodledata` klasörünü `/etc/fstab` ile kalıcı olarak bağlayalım:

```bash
sudo mkdir -p /var/moodledata
echo "192.168.1.87:/mnt/moodledata /var/moodledata nfs rw,relatime,hard,proto=tcp,_netdev 0 0" | sudo tee -a /etc/fstab
sudo mount -a

```

#### B. Scalelite-Docker Sunucusunda Bağlantı

Scalelite ana makinesinde, NFS üzerinden gelen yayınlanmış ve kuyruktaki kayıt yollarını host sistemine mount edelim:

```bash
sudo mkdir -p /mnt/scalelite-published
sudo mkdir -p /mnt/scalelite-spool

# /etc/fstab dosyasına ekleme
echo "192.168.1.87:/mnt/scalelite-recordings/var/bigbluebutton/published /mnt/scalelite-published nfs4 rw,relatime,vers=4.2,hard,proto=tcp,_netdev 0 0" | sudo tee -a /etc/fstab
echo "192.168.1.87:/mnt/scalelite-recordings/var/bigbluebutton/spool /mnt/scalelite-spool nfs4 rw,relatime,vers=4.2,hard,proto=tcp,_netdev 0 0" | sudo tee -a /etc/fstab

sudo mount -a

```

---

### 🚨 NFS Yapılandırmasında Dikkat Edilmesi Gereken Ölümcül Kurallar

1. **`localcache` Asla NFS Üzerinde Tutulmamalıdır:** Moodle, her sayfa isteğinde binlerce küçük cache, session ve `.mustache` şablon dosyası üretir. NFS ağ protokolü bu mikro dosya kilitlerine (file locking) yetişemez, sistem **"errno=28 No space left on device"** veya **"Failed to write cache file"** uyarısıyla kilitlenir. Çözüm olarak `config.php` içerisinde `$CFG->localcachedir` değeri mutlaka sunucunun kendi yerel diski (SSD/HDD) olarak set edilmelidir (`/var/moodle_localcache`).
2. **Konteyner İçinden İzin Kontrolü:** İzinlerin doğruluğunu teyit etmek için her zaman `docker compose exec` ile ilgili konteynerin içinden yazma testi (`mkdir`) simüle edilmelidir.

---

## 📌 Bölüm 5: Scalelite ve BigBlueButton Kümesi

### 1️⃣ Docker Compose Düzenlemesi (`docker-compose.yml`)

NFS üzerinden sisteme bağladığımız yolları Docker ekosistemine ortak bir hacim (`&id002`) haritasıyla geçirmek için compose dosyamızı şu düzende kilitliyoruz:

```yaml
volumes: &id002
  - /opt/scalelite/log/scalelite:/app/log
  - /mnt/scalelite-spool:/var/bigbluebutton/spool
  - /mnt/scalelite-published:/var/bigbluebutton/published

scalelite-api:
  image: blindsidenetwks/scalelite:v1-bionic230-alpine-api
  restart: unless-stopped
  security_opt:
    - apparmor:unconfined # ⚠️ Proxmox LXC kilitlerini kırmak için kritik ayar
  volumes: *id002

scalelite-recording-importer:
  image: blindsidenetwks/scalelite:v1-bionic230-alpine-recording-importer
  restart: unless-stopped
  volumes: *id002

```

---

### 2️⃣ Proxmox VE ve AppArmor Güvenlik Duvarı Geçişi

Eğer Scalelite veya Docker sunucunuz bir **Proxmox LXC Konteyneri** üzerinde koşuyorsa, Docker'ın ağ katmanlarını inşa ederken AppArmor engeline takılması çok yüksek bir ihtimaldir (`Error response from daemon: AppArmor enabled on system but... Permission denied`).

Bu engeli aşmak için:

1. Proxmox ana makinesinin terminaline girin.
2. İlgili konteynerin konfigürasyon dosyasını açın: `nano /etc/pve/lxc/119.conf` (119 yerine kendi container ID'nizi yazın).
3. Dosyanın en altına şu iki satırı ekleyin:
```text
lxc.apparmor.profile: unconfined
lxc.cgroup2.devices.allow: a
lxc.cap.drop:

```


4. Proxmox arayüzünden konteyneri **Sert Kapatıp (Shutdown/Stop)** ardından yeniden başlatın (**Cold Start**). Sadece `restart` komutu bu ayarı dinamik okuyamaz.

---

## 🔧 Sorun Giderme İpuçları

### 🟥 Problem: `Failed to import recording: Permission denied @ dir_s_mkdir`

* **Neden olur?** Scalelite Importer konteyneri (`UID 2000`), yeni gelen kaydı işledikten sonra yayınlanan kayıtlar klasörüne (`/var/bigbluebutton/published`) yazmaya çalışıyor ama NFS diski üzerinde yazma yetkisi yok.
* **Kesin Çözüm:** NFS sunucusuna giderek ham klasörün yetkilerini güncelleyin:
```bash
sudo chown -R 2000:2000 /mnt/scalelite-recordings/var/bigbluebutton/published
sudo chmod -R 777 /mnt/scalelite-recordings/var/bigbluebutton/published

```



### 🟥 Problem: Moodle’da `codingerror - File store path does not exist`

* **Neden olur?** Moodle web sunucusu (`www-data`), `/var/moodledata` altında dinamik olarak cache klasörü açamıyor ya da NFS diski anlık olarak salt-okunur (`read-only`) moda düştü.
* **Kesin Çözüm:** Moodle sunucu terminalinde dizinleri elinizle hazırlayıp izin verin:
```bash
sudo mkdir -p /var/moodledata/cache /var/moodledata/localcache /var/moodledata/muc
sudo chown -R www-data:www-data /var/moodledata
sudo chmod -R 775 /var/moodledata
sudo -u www-data php /var/www/moodle/admin/cli/purge_caches.php

```



---

**Son Güncelleme:** Haziran 2026

**Versiyon:** 4.0 (NFS Mimari Entegrasyonu ve UID/GID Protokolleri Eklendi)
