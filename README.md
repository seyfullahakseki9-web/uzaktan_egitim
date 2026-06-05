# 🎓 Yüksek Erişilebilir (HA) Uzaktan Eğitim Altyapısı

Binlerce öğrenciye aynı anda hizmet verebilmek için tasarlanmış, **Yüksek Erişilebilirlik (HA)** ve **Ölçeklenebilirlik** prensiplerine dayalı kapsamlı bir Moodle altyapısının kurulumu rehberidir.

---

## 📋 İçindekiler
1. [Sistem Mimarisi](#sistem-mimarisi)
2. [Bölüm 1: Moodle Uygulama Sunucularının Kurulumu](#bölüm-1-moodle-uygulama-sunucularının-kurulumu)
3. [Bölüm 2: HAProxy ile Yük Dengeleme](#bölüm-2-haproxy-ile-yük-dengeleme)
4. [Bölüm 3: Scalelite ve BigBlueButton Kümesi](#bölüm-3-scalelite-ve-bigbluebutton-kümesi)
5. [Sorun Giderme İpuçları](#sorun-giderme-ipuçları)

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
    ┌────────┐ ┌────────┐ ┌────────┐
    │PostgreSQL │ Redis  │  NFS    │
    │Veritabanı │Session │ Dosyalar│
    │192.168.1.100│192.168.1.185│192.168.1.50│
    └────────┘ └────────┘ └────────┘

┌─────────────────────────────────────────────┐
│ Canlı Dersler (BigBlueButton Kümesi)       │
├─────────────────────────────────────────────┤
│  Scalelite Yük Dengeleyici                  │
│  (192.168.1.60)                             │
│         │                                    │
│    ┌────┴────┬─────────┐                   │
│    ▼         ▼         ▼                   │
│  BBB-1    BBB-2    BBB-N   ... N adet BBB │
│  Node     Node     Node      Sunucusu      │
└─────────────────────────────────────────────┘
```

### 🖥️ Bileşen Özeti

| Bileşen | Sayı | Görev | IP Örneği |
|---------|------|-------|----------|
| **HAProxy** | 1 | SSL Sonlandırması, Trafik Dağıtımı | 192.168.1.10 |
| **Moodle Node** | 10 | Eğitim Platformu (Nginx + PHP) | 192.168.1.11-20 |
| **PostgreSQL** | 1 | Veritabanı | 192.168.1.100 |
| **Redis** | 1 | Oturum ve Önbellek | 192.168.1.185 |
| **NFS**(vm) | 1 | Ortak Dosya Depolama | 192.168.1.50 |
| **Scalelite** | 1 | BBB Yük Dengeleyici | 192.168.1.60 |
| **BigBlueButton**(vm) | N | Canlı Ders Sunucuları | 192.168.1.70+ |

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

Moodle'ın tüm sunucularda aynı dosyalara (ödevler, kütüphane vb.) erişebilmesi için NFS kullanıyoruz.

```bash
# NFS bağlanacak klasörü oluştur
sudo mkdir -p /var/moodledata

# NFS sunucusunu bağla
# ⚠️ 192.168.1.50 kısmını kendi NFS sunucunuzun IP'si ile değiştirin
sudo mount 192.168.1.50:/export/moodledata /var/moodledata

# Sunucu yeniden başlatılsa da otomatik bağlansın diye fstab dosyasına ekle
echo "192.168.1.50:/export/moodledata /var/moodledata nfs defaults,_netdev 0 0" | \
  sudo tee -a /etc/fstab
```

**Güvenlik Ayarlaması (KRITIK):**

Moodle web sunucusu (www-data kullanıcısı) bu klasöre yazabilir olmalıdır:

```bash
# Klasöre www-data sahipliğini ver
sudo chown -R www-data:www-data /var/moodledata

# Uygun izinleri ayarla
sudo chmod -R 755 /var/moodledata

# İzinleri doğrula
ls -ld /var/moodledata
# Çıktı: drwxr-xr-x 3 www-data www-data şeklinde olmalı
```

**NFS bağlantısını test et:**
```bash
# NFS bağlı mı kontrol et
df -h | grep moodledata
# Test dosyası oluştur
sudo -u www-data touch /var/moodledata/test.txt
# Başarılı olursa klasörden sil
sudo -u www-data rm /var/moodledata/test.txt
```

---

### 3️⃣ Moodle Kodlarının Hazırlanması

```bash
# Moodle için klasör oluştur
sudo mkdir -p /var/www/moodle

# Moodle kodlarını indir (resmi kaynaktan)
# Not: İlk kurulumda Moodle'ı git ile klonlayabilir veya indirilen dosyaları kopyalayabilirsiniz
cd /tmp
git clone https://github.com/moodle/moodle.git --depth 1 --branch MOODLE_401_STABLE
sudo cp -r moodle/* /var/www/moodle/

# Moodle dizininin sahibini ayarla
sudo chown -R www-data:www-data /var/www/moodle
sudo chmod -R 755 /var/www/moodle
```

---

### 4️⃣ Moodle Konfigürasyon Dosyası (config.php)

Kümelenmiş mimaride çalışmak için özel bir config.php dosyası oluşturmalıyız:

```bash
sudo nano /var/www/moodle/config.php
```

Aşağıdaki içeriği dosyaya yapıştırın (IP adreslerini kendi ağınıza göre düzenleyin):

```php
<?php
// ============================================================================
// Moodle Yüksek Erişilebilirlik (HA) Kümesi Konfigürasyonu
// ============================================================================

unset($CFG);
global $CFG;
$CFG = new stdClass();

// ============================================================================
// 1. VERITABANI AYARLARI (Ortak PostgreSQL Sunucusu)
// ============================================================================
$CFG->dbtype    = 'pgsql';           // Veritabanı türü
$CFG->dblibrary = 'native';          // Native PostgreSQL sürücüsü
$CFG->dbhost    = '192.168.1.100';   // ⚠️ PostgreSQL sunucusu IP'sini yazın
$CFG->dbname    = 'moodle';          // Veritabanı adı
$CFG->dbuser    = 'moodleuser';      // Veritabanı kullanıcısı
$CFG->dbpass    = 'VERITABANI_SIFRENIZ'; // ⚠️ Güvenli şifre belirleyin
$CFG->prefix    = 'mdl_';            // Tablo ön eki

// ============================================================================
// 2. WEB VE KLASÖR AYARLARI
// ============================================================================
$CFG->wwwroot   = 'https://moodle.argeyazilim.tr'; // ⚠️ Kendi domain'inizi yazın
$CFG->dataroot  = '/var/moodledata';  // NFS'ten gelen ortak klasör
$CFG->admin     = 'admin';            // Admin paneli URL'i
$CFG->directorypermissions = 0770;    // Klasör izinleri

// ============================================================================
// 3. REVERSE PROXY AYARLARI (HAProxy Arkasında Olmak İçin)
// ============================================================================
// HAProxy HTTPS bağlantısını HTTP'ye çevirdiği için bunu belirtmemiz gerekir
$CFG->sslproxy = true;        // HAProxy'nin SSL sonlandırmasını kabullen
$CFG->reverseproxy = true;    // Reverse proxy arkasında olduğumuzu söyle

// ============================================================================
// 4. OTURUM YÖNETİMİ (Redis Kullanarak - Tüm Sunucularda Aynı)
// ============================================================================
// Kullanıcıların oturumu kapatılmaması için oturum bilgileri merkezi Redis'te tutulur
$CFG->session_handler_class = '\core\session\redis';
$CFG->session_redis_host = '192.168.1.185';      // ⚠️ Redis sunucusu IP'sini yazın
$CFG->session_redis_port = 6379;                 // Redis port
$CFG->session_redis_auth = 'REDIS_SIFRENIZ';     // ⚠️ Redis şifresi (varsa)
$CFG->session_redis_prefix = 'moodle_session_';  // Oturum ön eki
$CFG->session_redis_acquire_lock_timeout = 120;  // Kilit bekleme süresi
$CFG->session_redis_lock_expire = 7200;          // Kilit süresi (2 saat)

// ============================================================================
// 5. ÖNBELLEK YÖNETİMİ (İsteğe Bağlı - Performans için)
// ============================================================================
// $CFG->cachestore_redis_server = '192.168.1.185:6379';
// $CFG->cachestore_redis_auth = 'REDIS_SIFRENIZ';

// ============================================================================
// 6. İLETİŞİM AYARLARI (E-posta)
// ============================================================================
$CFG->smtphosts = 'smtp.example.com:25';  // ⚠️ SMTP sunucunuzu yazın
$CFG->noreplyaddress = 'noreply@moodle.argeyazilim.tr';
$CFG->supportemail = 'support@argeyazilim.tr';

// ============================================================================
// 7. GÜVENLİK AYARLARI
// ============================================================================
// Sadece HTTPS ile erişime izin ver
$CFG->disableupdatenotifications = true; // İç ağda olduğu için güncelleme bildirimleri kapat

// ============================================================================
// 8. DEBUG AYARLARI (Üretim Ortamında Kapalı Kalmalı)
// ============================================================================
$CFG->debug = DEBUG_NONE;  // Üretim ortamında sorunları tespit etmek için ERROR'a değiştirebilirsiniz
$CFG->debugdisplay = false;
$CFG->debugstringids = false;

// Moodle kurulum dosyasını başlat
require_once(__DIR__ . '/lib/setup.php');

// Kurulum bitişinde bu satırı açın (Kurulum sonrası)
// require_once(__DIR__ . '/lib/setup.php');
```

---

### 5️⃣ Nginx Konfigürasyonu

HAProxy HAProxy arkasında çalıştığı için, Nginx sadece port 80'de (HTTP) dinleyecektir. SSL (HTTPS) şifrelemesi HAProxy tarafından yapılır.

```bash
sudo nano /etc/nginx/sites-available/moodle
```

Aşağıdaki konfigürasyonu yapıştırın:

```nginx
# Moodle Sunucusu Konfigürasyonu
server {
    # Port 80'de dinle (HAProxy'den trafik gelecek)
    listen 80 default_server;
    listen [::]:80 default_server;
    
    # Domain adı
    server_name moodle.argeyazilim.tr _;
    
    # Moodle klasörü
    root /var/www/moodle;
    index index.php;
    
    # Geniş dosya yükleme desteği (100MB'ye kadar)
    client_max_body_size 100M;
    
    # ============================================================================
    # HAProxy'den Gelen Gerçek IP'yi Almak (Logging için)
    # ============================================================================
    # Not: HAProxy IP'sini yazın (Ör: 192.168.1.10)
    set_real_ip_from 192.168.1.10;
    real_ip_header X-Forwarded-For;
    
    # ============================================================================
    # Tüm İstekler
    # ============================================================================
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    # ============================================================================
    # PHP Dosyaları (index.php gibi)
    # ============================================================================
    location ~ [^/]\.php(/|$) {
        # FastCGI yapılandırması
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        
        # Script dosyasının tam yolunu ilet
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        
        # HAProxy'den gelen bilgileri PHP'ye ilet
        fastcgi_param HTTP_X_FORWARDED_FOR $proxy_add_x_forwarded_for;
        fastcgi_param HTTP_X_FORWARDED_PROTO $http_x_forwarded_proto;
        
        # Zaman aşımı ayarları (büyük dosyalar için)
        fastcgi_connect_timeout 60s;
        fastcgi_send_timeout 300s;
        fastcgi_read_timeout 300s;
        
        include fastcgi_params;
    }
    
    # ============================================================================
    # Gizli Dosyalara Erişimi Engelle
    # ============================================================================
    # .htaccess dosyalarına erişim istemeyin
    location ~ /\.ht {
        deny all;
    }
    
    # .env dosyasına erişim engelle
    location ~ /\.env {
        deny all;
    }
    
    # Moodle'ın private dosyalarına erişim engelle
    location ~ ^/\.git {
        deny all;
    }
}
```

Konfigürasyonu etkinleştir:

```bash
# Konfigürasyonu sites-enabled klasörüne bağla
sudo ln -s /etc/nginx/sites-available/moodle /etc/nginx/sites-enabled/

# Varsayılan konfigürasyonu devre dışı bırak (sorun yaratabilir)
sudo rm /etc/nginx/sites-enabled/default 2>/dev/null || true

# Syntax kontrolü yap
sudo nginx -t
# Çıktı: "test is successful" görmelisiniz

# Nginx'i yeniden başlat
sudo systemctl restart nginx
sudo systemctl enable nginx  # Önyükleme sırasında otomatik başlat
```

---

### 6️⃣ PHP-FPM Ayarlaması

PHP'nin optimum performans sunması için ayarları yapılandır:

```bash
sudo nano /etc/php/8.3/fpm/php.ini
```

Aşağıdaki satırları bulup değiştir:

```ini
; Dosya yükleme limiti
upload_max_filesize = 100M

; POST veri limiti
post_max_size = 100M

; Maksimum yürütme süresi
max_execution_time = 300

; Bellek limiti
memory_limit = 256M

; Zaman dilimi
date.timezone = Europe/Istanbul
```

PHP-FPM'i yeniden başlat:

```bash
sudo systemctl restart php8.3-fpm
```

---

### 7️⃣ Cron Görevi Ayarı (ÖNEMLİ: Sadece Node-1'de)

Moodle, cron görevi aracılığıyla veritabanını günceller ve e-postaları gönderir. **Kümelenmiş mimaride bu görev SADECE bir sunucuda çalışmalıdır** (aksi takdirde e-postalar birden fazla gönderilir).

**SADECE Node-1 (Master Node) üzerinde:**

```bash
# www-data kullanıcısı için crontab'ı aç
sudo crontab -u www-data -e

# Aşağıdaki satırı ekle (dakikada bir çalışması için):
```

```bash
* * * * * /usr/bin/php /var/www/moodle/admin/cli/cron.php >/dev/null 2>&1
```

**Diğer 9 sunucuda (Node-2 ile Node-10) cron görevi oluşturmayın!**

---

### 8️⃣ Moodle Kurulumunu Tamamlama

Node-1'de Moodle veritabanını initialize et:

```bash
# Moodle komut dosyasını çalıştır
sudo -u www-data php /var/www/moodle/admin/cli/install_database.php \
  --lang=tr \
  --adminpass=GUCLU_YONETİCİ_SIFRENIZ \
  --agree-license
```

Kurulum başarılı olursa, tarayıcıdan `http://192.168.1.11` (Node-1'in IP'si) adresine gittiğinizde Moodle anasayfasını görebilirsiniz.

---

### 9️⃣ Yapılandırmayı Diğer Sunuculara Kopyalama

Node-1 başarıyla kurulduktan sonra, Node-2'den Node-10'a kadar olan sunucularda aynı adımları takip edin, ancak veritabanı kurulumu yapmanız yeterli değil - aşağıdaki adımları yapın:

**Her sunucuda (Node-2 ile Node-10):**

```bash
# 1. Gerekli yazılımları kur (1. adım)
# 2. NFS'i bağla (2. adım)
# 3. Moodle kodlarını kopyala (3. adım)
# 4. config.php dosyasını kopyala (4. adım)
# 5. Nginx konfigürasyonunu kopyala (5. adım)
# 6. PHP ayarlarını kopyala (6. adım)

# NOT: 7. adımda (Cron) hiçbir şey yapmayın!

# Test et
curl http://192.168.1.12/login/index.php
```

---

### 🟣 Node-1'den Diğer Sunuculara Kopyalama (İsteğe Bağlı Hızlı Yöntem)

Eğer Node-1'de her şey hazırsa, aşağıdaki komutu Node-1'de çalıştırarak konfigürasyonları Node-2 ile Node-10'a kopyalayabilirsiniz:

```bash
# Node-1'den diğer sunuculara config dosyalarını kopyala (SSH gerekli)
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

**Amaç:** İnternet'ten gelen HTTPS trafiğini karşılayıp, arkadaki Moodle sunucularına HTTP olarak dağıtmak.

---

### 1️⃣ HAProxy Kurulumu

HAProxy sunucusuna SSH ile bağlan ve kur:

```bash
# Sistem paketlerini güncelle
sudo apt update && sudo apt upgrade -y

# HAProxy'i kur
sudo apt install -y haproxy

# Sürümü kontrol et
haproxy -v
# Minimum v2.2 olmalı
```

---

### 2️⃣ SSL Sertifikasının Hazırlanması

**Seçenek A: Let's Encrypt'ten Sertifika Alıyorsanız**

```bash
# Certbot ve Nginx eklentisini kur
sudo apt install -y certbot python3-certbot-nginx

# Sertifika al
sudo certbot certonly --standalone \
  -d moodle.argeyazilim.tr \
  -d www.moodle.argeyazilim.tr \
  --email your@email.com \
  --agree-tos
```

**Seçenek B: Sertifikayı Zaten Aldıysanız**

Sertifika ve özel anahtarı birleştir:

```bash
# HAProxy için sertifika klasörü oluştur
sudo mkdir -p /etc/haproxy/certs

# Sertifika ve özel anahtarı birleştir
# ⚠️ Dosya yollarını kendi sertifikalarınızla değiştirin
sudo bash -c 'cat /etc/letsencrypt/live/moodle.argeyazilim.tr/fullchain.pem \
  /etc/letsencrypt/live/moodle.argeyazilim.tr/privkey.pem > \
  /etc/haproxy/certs/moodle.argeyazilim.tr.pem'

# Sertifika izinlerini ayarla (sadece root okuyabilir)
sudo chmod 600 /etc/haproxy/certs/moodle.argeyazilim.tr.pem

# Sertifikanın geçerli olup olmadığını kontrol et
sudo openssl x509 -in /etc/haproxy/certs/moodle.argeyazilim.tr.pem -text -noout | grep -A 2 "Validity"
```

---

### 3️⃣ HAProxy Konfigürasyonu

HAProxy'nin ana yapılandırma dosyasını düzenle:

```bash
# Yedek kopyasını al
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak

# Dosyayı aç
sudo nano /etc/haproxy/haproxy.cfg
```

**Eski içeriği silin ve aşağıdakini yapıştırın:**

```haproxy
# ============================================================================
# HAProxy Konfigürasyonu - Moodle Yüksek Erişilebilirlik
# ============================================================================

global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    
    # Maksimum bağlantı sayısı
    maxconn 4096
    
    # ============================================================================
    # Modern SSL/TLS Ayarları (Güvenlik İçin)
    # ============================================================================
    # Güvenli şifreler (eski ve zayıf şifreleri engelle)
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    
    # Eski SSL/TLS versiyonlarını devre dışı bırak
    ssl-default-bind-options no-sslv3 no-tlsv10 no-tlsv11 no-tls-tickets

# ============================================================================
# VARSAYILAN AYARLAR
# ============================================================================
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

# ============================================================================
# 1. HTTP İsteklerini HTTPS'e Yönlendir (Port 80)
# ============================================================================
frontend http_front
    bind *:80
    mode http
    
    # Tüm HTTP isteklerini HTTPS'e zorla
    # Ör: http://moodle.argeyazilim.tr → https://moodle.argeyazilim.tr
    redirect scheme https code 301 if !{ ssl_fc }

# ============================================================================
# 2. HTTPS Trafiğini Karşıla (Port 443)
# ============================================================================
frontend https_front
    bind *:443 ssl crt /etc/haproxy/certs/moodle.argeyazilim.tr.pem
    mode http
    
    # Gelen trafiğin gerçek kaynağını izle
    option forwardfor
    
    # Moodle'a iletmesi gereken başlık bilgileri
    http-request set-header X-Forwarded-Proto https
    http-request set-header X-Forwarded-Port 443
    
    # Tüm trafiği moodle_cluster backend'ine yönlendir
    default_backend moodle_cluster

# ============================================================================
# 3. MOODLE SUNUCULARI (Backend)
# ============================================================================
backend moodle_cluster
    mode http
    
    # ============================================================================
    # Yük Dağıtım Algoritması
    # ============================================================================
    # leastconn: En az bağlantısı olan sunucuya gönder (önerilen)
    # roundrobin: Sırasıyla dağıt
    balance leastconn
    
    # Moodle sunucularına iletmesi gereken başlık bilgileri
    option forwardfor
    http-request set-header X-Real-IP %[src]
    
    # ============================================================================
    # Sağlık Kontrolü (Health Check)
    # ============================================================================
    # Her 2 saniyede bir sunucuların sağlığını kontrol et
    option httpchk GET /login/index.php HTTP/1.0
    http-check expect status 200
    
    # Sunucu ayakta kalma parametreleri
    default-server inter 2000 fall 3 rise 2
    
    # ============================================================================
    # 10 Adet Moodle Sunucusu
    # ============================================================================
    # ⚠️ IP adreslerini kendi ağınızla değiştirin
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

# ============================================================================
# 4. İSTATİSTİK PANELİ (İsteğe Bağlı - Önerilir)
# ============================================================================
# HAProxy'nin web tabanlı yönetim paneli
listen stats
    bind *:8404
    stats enable
    stats uri /haproxy_stats
    stats refresh 10s
    stats auth admin:GUCLU_BIR_SIFRE_YAZIN
    mode http
```

---

### 4️⃣ HAProxy Yapılandırmasını Test ve Başlatma

```bash
# Syntax kontrol
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
# Çıktı: "Configuration file is valid" görmelisiniz

# Yapılandırma geçerliyse servisi yeniden başlat
sudo systemctl restart haproxy
sudo systemctl status haproxy

# Önyükleme sırasında otomatik başlat
sudo systemctl enable haproxy
```

---

### 5️⃣ HAProxy'i Test Etme

**Test 1: Basit bağlantı testi**
```bash
# HAProxy sunucusundan test et
curl -I http://localhost
# Çıktı: HTTP/1.1 301 (HTTPS yönlendirmesi)

curl -I https://localhost -k
# Çıktı: HTTP/1.1 302 (Moodle yönlendirmesi)
```

**Test 2: İstatistik Panelini Kontrol Et**

Tarayıcıdan açın:
```
http://HAPROXY_IP_ADRESI:8404/haproxy_stats
```

Kullanıcı adı: `admin`
Şifre: HAProxy konfigürasyonunda belirlediğiniz şifre

Panel'de 10 sunucunun tümü yeşil (UP) olmalıdır.

**Test 3: Son kullanıcı testi**

Tarayıcıdan açın:
```
https://moodle.argeyazilim.tr
```

---

## 📌 Bölüm 3: Scalelite ve BigBlueButton Kümesi

Canlı dersler (video konferans) için binlerce öğrenciye eşzamanlı hizmet vermek üzere BigBlueButton sunucularını bir havuzda yönetiyoruz.

**Not:** Bu bölüm ileri seviye bir kurulumun ön bilgisini gerektirir. Detaylı talimatları için [Scalelite Resmi Dokümantasyonu](https://github.com/blindsidenetworks/scalelite)nu inceleyiniz.

---

### 1️⃣ Scalelite İçin NFS Ayarı (Kayıt Depolama)

Scalelite sunucusunda:

```bash
# Kayıt klasörlerini oluştur
sudo mkdir -p /var/bigbluebutton/spool
sudo mkdir -p /var/bigbluebutton/published
sudo mkdir -p /var/bigbluebutton/unpublished

# NFS sunucusundan bağla (NFS sunucusunuzun IP'sini yazın: Ör 192.168.1.50)
sudo mount 192.168.1.50:/export/bbb_spool /var/bigbluebutton/spool
sudo mount 192.168.1.50:/export/bbb_published /var/bigbluebutton/published
sudo mount 192.168.1.50:/export/bbb_unpublished /var/bigbluebutton/unpublished

# Otomatik bağlanma için fstab dosyasını güncelle
echo "192.168.1.50:/export/bbb_spool /var/bigbluebutton/spool nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
echo "192.168.1.50:/export/bbb_published /var/bigbluebutton/published nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
echo "192.168.1.50:/export/bbb_unpublished /var/bigbluebutton/unpublished nfs defaults,_netdev 0 0" | sudo tee -a /etc/fstab

# İzinleri ayarla
sudo chmod -R 755 /var/bigbluebutton/
```

---

### 2️⃣ Scalelite Ortam Değişkenleri (.env)

Scalelite sunucusunda `/var/www/scalelite/.env` dosyasını oluştur:

```bash
sudo nano /var/www/scalelite/.env
```

```bash
# ============================================================================
# Scalelite Konfigürasyonu
# ============================================================================

# URL ve Kimlik Tanımlaması
URL_HOST=scalelite.argeyazilim.tr
SCALELITE_TAG=scalelite.argeyazilim.tr

# ============================================================================
# Güvenlik Anahtarları
# ============================================================================
# SECRET_KEY_BASE: Rastgele 64 karakter oluştur:
# Komut: openssl rand -hex 32
SECRET_KEY_BASE=BURAYA_OLUSTURULAN_64_KARAKTERLIK_ANAHTAR

# LOADBALANCER_SECRET: Moodle'ın Scalelite'a bağlanacağı şifre
LOADBALANCER_SECRET=BURAYA_MOODLE_ICIN_GUCLU_SIFRE

# ============================================================================
# Veritabanı Bağlantısı (PostgreSQL)
# ============================================================================
# Format: postgres://kullanici:sifre@SUNUCU_IP/veritabani_adi
DATABASE_URL=postgres://scalelite:SIFRE@192.168.1.100/scalelite_production

# ============================================================================
# Oturum ve Önbellek (Redis)
# ============================================================================
# Format: redis://SUNUCU_IP:PORT/0
REDIS_URL=redis://192.168.1.185:6379/0

# ============================================================================
# Ortam
# ============================================================================
RAILS_ENV=production

# ============================================================================
# BBB Yapılandırması (Opsiyonel)
# ============================================================================
BIGBLUEBUTTON_SECRET=shared_secret
```

---

### 3️⃣ Scalelite Poller Servisi (Opsiyonel - Durumu İzlemek İçin)

Scalelite'ın BigBlueButton sunucularının sağlığını kontrol etmesi için:

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

# 60 saniyede bir sağlık kontrolünü çalıştır
ExecStart=/bin/bash -c 'while true; do RAILS_ENV=production bundle exec rake poll:all; sleep 60; done'

Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Servisi başlat:

```bash
sudo systemctl daemon-reload
sudo systemctl enable scalelite-poller
sudo systemctl start scalelite-poller
```

---

### 4️⃣ BigBlueButton Sunucularını Scalelite'a Ekleme

Her BigBlueButton sunucusu için:

```bash
# BBB sunucusundan API gizli anahtarını al
sudo bbb-conf --secret
# Çıktı:
# URL: https://bbb-node-01.example.com/bigbluebutton/api
# Secret: 8e0a1be...
```

Scalelite sunucusunda, BBB sunucularını havuza ekle:

```bash
# BBB Node-1'i ekle
cd /var/www/scalelite
RAILS_ENV=production bundle exec rake servers:add[\
https://bbb-node-01.argeyazilim.tr/bigbluebutton/api,\
SUNUCU_1_SECRET_KODU\
]

# BBB Node-2'yi ekle
RAILS_ENV=production bundle exec rake servers:add[\
https://bbb-node-02.argeyazilim.tr/bigbluebutton/api,\
SUNUCU_2_SECRET_KODU\
]

# Daha fazla BBB sunucusu varsa tekrar et...
```

Eklenen sunucuları listele:

```bash
RAILS_ENV=production bundle exec rake servers
```

Çıktı örneği:
```
1. https://bbb-node-01.argeyazilim.tr/bigbluebutton/api
2. https://bbb-node-02.argeyazilim.tr/bigbluebutton/api
...
```

Sunucuları aktifleştir (1, 2, 3... şeklinde ID'lerle):

```bash
RAILS_ENV=production bundle exec rake servers:enable[1]
RAILS_ENV=production bundle exec rake servers:enable[2]
```

---

### 5️⃣ Moodle'da Scalelite Konfigürasyonu

Herhangi bir Moodle sunucusuna yönetici hesabıyla giriş yap:

1. **Site Yönetimi** → **Eklentiler** → **Eklenti Yönetimi** → **BigBlueButton** ara
2. **Ayarlar** sekmesine tıkla
3. Aşağıdaki bilgileri gir:

| Ayar | Değer |
|------|-------|
| **BigBlueButton Server URL** | `https://scalelite.argeyazilim.tr/bigbluebutton/api` |
| **Shared Secret** | Scalelite `.env` dosyasındaki `LOADBALANCER_SECRET` |

4. **Kaydet** butonuna tıkla

Artık Moodle'dan oluşturulan her canlı ders, Scalelite tarafından analiz edilerek en uygun BBB sunucusunda başlatılır.

---

## 🔧 Sorun Giderme İpuçları

### 🟥 Problem 1: Moodle Sunucuları HAProxy'de DOWN Gösteriliyor

```bash
# 1. Sunucunun çalışıp çalışmadığını kontrol et
ping 192.168.1.11

# 2. Nginx çalışıyor mu?
ssh user@192.168.1.11 "systemctl status nginx"

# 3. Sağlık kontrolünün URL'sini manuel test et
curl http://192.168.1.11/login/index.php

# 4. HAProxy loglarını kontrol et
sudo tail -f /var/log/haproxy.log
```

### 🟥 Problem 2: Kullanıcılar Oturumdan Düşüyor

```bash
# Redis bağlantısını test et
redis-cli -h 192.168.1.185
> ping
# Çıktı: PONG

# Moodle sunucusunun config.php'sini kontrol et
grep -A 5 "session_redis" /var/www/moodle/config.php
```

### 🟥 Problem 3: Dosya Yükleme Başarısız Oluyor

```bash
# NFS bağlantısını kontrol et
df -h /var/moodledata

# İzinleri kontrol et
ls -ld /var/moodledata
# Çıktı: drwxr-xr-x www-data www-data şeklinde olmalı

# NFS sunucusundan test yap
touch /var/moodledata/test.txt
rm /var/moodledata/test.txt
```

### 🟥 Problem 4: SSL Sertifikası Hatası

```bash
# Sertifikanın geçerlilik tarihini kontrol et
openssl x509 -in /etc/haproxy/certs/moodle.argeyazilim.tr.pem \
  -noout -dates

# Let's Encrypt sertifikasını yenile
sudo certbot renew --dry-run

# HAProxy'ye yeni sertifikayı sağla
sudo bash -c 'cat /etc/letsencrypt/live/moodle.argeyazilim.tr/fullchain.pem \
  /etc/letsencrypt/live/moodle.argeyazilim.tr/privkey.pem > \
  /etc/haproxy/certs/moodle.argeyazilim.tr.pem'

sudo systemctl restart haproxy
```

---

## 📚 Ek Kaynaklar

- [Moodle Resmi Dokümantasyonu](https://docs.moodle.org)
- [Moodle Küme Kurulumu](https://docs.moodle.org/en/Setting_up_a_cluster)
- [HAProxy Dokümantasyonu](http://www.haproxy.org/)
- [BigBlueButton](https://docs.bigbluebutton.org/)
- [Scalelite](https://github.com/blindsidenetworks/scalelite)
- [PostgreSQL Dokümantasyonu](https://www.postgresql.org/docs/)
- [Redis Dokümantasyonu](https://redis.io/documentation)

---

## 💬 Katkıda Bulunma

Bu rehbere katkıda bulunmak istiyorsanız, lütfen [GitHub Issues](https://github.com) üzerinden feedback veya pull request gönderin.

---

## 📄 Lisans

Bu dokümantasyon Creative Commons Attribution 4.0 International License altında yayımlanmıştır.

---

**Son Güncelleme:** Haziran 2026
**Versiyon:** 2.0 (Geliştirilmiş ve Açıklanmış Sürüm)
