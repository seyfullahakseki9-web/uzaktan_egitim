# 🎓 Yüksek Erişilebilir (HA) Uzaktan Eğitim Altyapısı

Bu depo, on binlerce öğrenciye aynı anda hizmet verebilmek amacıyla tasarlanmış, **High Availability (Yüksek Erişilebilirlik)** prensiplerine dayalı devasa bir uzaktan eğitim altyapısının kurulumunu içerir.

### 🏗️ Sistem Mimarisi
* **Yük Dengeleyici (Web):** 1x HAProxy (İstemci trafiğini Moodle sunucularına dağıtır)
* **Uygulama Sunucuları:** 10x Moodle Node (Nginx + PHP 8.3)
* **Veritabanı:** 1x PostgreSQL Sunucusu
* **Oturum ve Önbellek:** 1x Redis Sunucusu
* **Ortak Depolama:** 1x NFS Sunucusu (Tüm Moodle sunucuları `moodledata` dizinini buradan okur)
* **Canlı Ders Altyapısı:** Scalelite (Yük Dengeleyici) + BigBlueButton Kümesi

---

## 📌 Bölüm 1: Moodle Uygulama Sunucularının (Node) Kurulumu

Bu adımlar, HAProxy arkasında çalışacak olan 10 adet Moodle sunucusundan **ilkini (Master Node)** kurmak için hazırlanmıştır. İlk sunucu başarıyla yapılandırıldıktan sonra, bu sunucu klonlanarak (veya konfigürasyonu kopyalanarak) diğer 9 sunucuya dağıtılabilir.

### 🛠️ 1. Gerekli Paketlerin Kurulması

Uygulama sunucusunda (Node) Nginx, PHP 8.3 ve NFS istemcisi kurulu olmalıdır:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y nginx nfs-common redis-tools postgresql-client
sudo apt install -y php8.3-fpm php8.3-pgsql php8.3-curl php8.3-gd php8.3-intl php8.3-mbstring php8.3-soap php8.3-xml php8.3-xmlrpc php8.3-zip php8.3-redis
📂 2. NFS (Ortak Depolama) Bağlantısının Yapılması
10 Moodle sunucusunun aynı dosyalara (yüklenen ödevler, videolar vb.) erişebilmesi için moodledata klasörü NFS sunucusundan (Örn: 192.168.1.50) mount edilmelidir.
Bash
# Klasörü oluştur
sudo mkdir -p /var/moodledata

# NFS sunucusunu bağla
sudo mount 192.168.1.50:/export/moodledata /var/moodledata

# Yeniden başlatmalarda otomatik bağlanması için fstab dosyasına ekle:
echo "192.168.1.50:/export/moodledata /var/moodledata nfs defaults 0 0" | sudo tee -a /etc/fstab
Kritik Güvenlik Ayarı: Klasör sahipliğinin ve izinlerinin Moodle'ın çalışabileceği şekilde ayarlanması şarttır. Aksi takdirde Permission denied hataları alınır:
Bash
sudo chown -R www-data:www-data /var/moodledata
sudo chmod -R 775 /var/moodledata
⚙️ 3. Moodle Kodlarının Çekilmesi ve config.php Ayarları
Moodle dosyalarını /var/www/moodle dizinine yerleştirdikten sonra, küme mimarisine (Cluster) özel config.php dosyasını oluşturun:
PHP
<?php  // Moodle Cluster Configuration File

unset($CFG);
global $CFG;
$CFG = new stdClass();

// 1. Ortak Veritabanı Ayarları (PostgreSQL Sunucusu)
$CFG->dbtype    = 'pgsql';
$CFG->dblibrary = 'native';
$CFG->dbhost    = 'POSTGRESQL_SUNUCU_IP'; // Örn: 192.168.1.100
$CFG->dbname    = 'moodle';
$CFG->dbuser    = 'moodleuser';
$CFG->dbpass    = 'VERITABANI_SIFRENIZ';
$CFG->prefix    = 'mdl_';

// 2. Web, Proxy ve Dizin Ayarları
$CFG->wwwroot   = '[https://moodle.argeyazilim.tr](https://moodle.argeyazilim.tr)';
$CFG->dataroot  = '/var/moodledata'; // NFS üzerinden gelen ortak klasör
$CFG->admin     = 'admin';
$CFG->directorypermissions = 0770;

// HAProxy arkasında olduğumuz için Moodle'a SSL arkasında olduğunu belirtiyoruz
$CFG->sslproxy = true; 
$CFG->reverseproxy = true;

// 3. Ortak Oturum (Session) ve Önbellek Yönetimi (Redis)
// 10 sunucuda kullanıcıların oturumdan düşmemesi için Session'lar Redis'te tutulur
$CFG->session_handler_class = '\core\session\redis';
$CFG->session_redis_host = 'REDIS_SUNUCU_IP'; // Örn: 192.168.1.185
$CFG->session_redis_port = 6379;
$CFG->session_redis_auth = 'REDIS_SIFRENIZ';
$CFG->session_redis_prefix = 'moodle_session_';
$CFG->session_redis_acquire_lock_timeout = 120;
$CFG->session_redis_lock_expire = 7200;

require_once(__DIR__ . '/lib/setup.php');
🌐 4. Nginx Yapılandırması (Uygulama Sunucusu)
Bu sunucular HAProxy'nin arkasında olduğu için SSL (443) işlemi HAProxy üzerinde sonlandırılır (SSL Offloading). Nginx sadece 80 portundan gelen istekleri karşılayacak şekilde ayarlanır.
Bash
sudo nano /etc/nginx/sites-available/moodle
Nginx
server {
    listen 80;
    server_name moodle.argeyazilim.tr;
    root /var/www/moodle;
    index index.php;

    client_max_body_size 100M;

    # HAProxy'den gelen gerçek IP'yi almak için
    set_real_ip_from HAPROXY_IP_ADRESINIZ;
    real_ip_header X-Forwarded-For;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ [^/]\.php(/|$) {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
Bash
sudo ln -s /etc/nginx/sites-available/moodle /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
⚠️ 5. Cron Görevi (Sadece TEK Bir Sunucuda)
Moodle Cron görevi veritabanını günceller ve mailleri gönderir. Küme (Cluster) mimarilerinde çok kritik bir kural vardır: Cron görevi 10 sunucunun hepsinde DEĞİL, sadece belirlediğiniz 1 adet Master sunucuda çalışmalıdır (Aksi takdirde çakışmalar ve çift mail gönderimleri yaşanır).
Sadece Node-1 üzerinde şu işlemi yapın:
Bash
sudo crontab -u www-data -e
Bash
* * * * * /usr/bin/php /var/www/moodle/admin/cli/cron.php >/dev/null
🚀 6. Sistemi Kurma ve Diğer Sunuculara Dağıtma
Her şey hazır olduğuna göre, Moodle'ı veritabanına kurmak için Node-1 üzerinde terminalden kurulumu başlatın:
Bash
sudo -u www-data php /var/www/moodle/admin/cli/install_database.php --lang=tr --adminpass=SISTEM_SIFRESI --agree-license
Bu işlem bittikten sonra Node-1 sunucunuz hazırdır. Node-2'den Node-10'a kadar olan sunucularda; NFS'i bağlayın, Nginx'i kurun ve Moodle kodları ile config.php dosyasını birebir kopyalayın. Moodle Cluster'ınız HAProxy üzerinden trafik almaya hazırdır!

## 📌 Bölüm 2: HAProxy ile Yük Dengeleme (Load Balancing) ve SSL Sonlandırma

Bu bölümde, dış dünyadan `moodle.argeyazilim.tr` adresine gelen trafiği karşılayacak, güvenli bağlantıyı (HTTPS) sağlayacak ve arkadaki 10 adet Moodle sunucusuna (Node) yükü eşit olarak dağıtacak olan **HAProxy** kurulumu anlatılmaktadır.

Mimarimizde oturum yönetimi (Session) merkezi bir Redis sunucusunda tutulduğu için, HAProxy üzerinde "Sticky Session" (Kullanıcıyı sürekli aynı sunucuya gönderme) zorunluluğu yoktur. Bu sayede tam verimli bir Round-Robin (sıralı dağıtım) yapılabilir.

### 🛠️ 1. HAProxy Kurulumu

Yük dengeleyici sunucunuza SSH ile bağlanın ve HAProxy'i kurun:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y haproxy
🔒 2. SSL Sertifikasının Hazırlanması (SSL Offloading)
Uygulama sunucularını (Moodle Node'ları) SSL şifreleme/çözme yükünden kurtarmak için SSL işlemini HAProxy üzerinde sonlandırıyoruz (SSL Offloading).
Eğer sertifikanızı (Örn: Let's Encrypt / Certbot ile) aldıysanız, HAProxy'nin okuyabilmesi için fullchain.pem ve privkey.pem dosyalarını tek bir .pem dosyasında birleştirmemiz gerekir:
Bash
# HAProxy için sertifika dizini oluşturma
sudo mkdir -p /etc/haproxy/certs

# Sertifika ve Gizli Anahtarı birleştirme
sudo cat /etc/letsencrypt/live/moodle.argeyazilim.tr/fullchain.pem /etc/letsencrypt/live/moodle.argeyazilim.tr/privkey.pem | sudo tee /etc/haproxy/certs/moodle.argeyazilim.tr.pem > /dev/null

# İzinleri ayarlama
sudo chmod 600 /etc/haproxy/certs/moodle.argeyazilim.tr.pem
⚙️ 3. HAProxy Yapılandırması (haproxy.cfg)
HAProxy'nin ana yapılandırma dosyasını yedeğini alıp düzenlemeye başlayalım:
Bash
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
sudo nano /etc/haproxy/haproxy.cfg
Dosyanın içeriğini altyapımıza uygun olarak aşağıdaki gibi güncelleyin (IP adreslerini kendi ağ yapınıza göre düzenleyin):
Kod snippet'i
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    
    # Modern SSL/TLS ayarları
    ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
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

# 1. HTTP'den HTTPS'e Yönlendirme (Frontend 80)
frontend http_front
    bind *:80
    # Gelen tüm HTTP isteklerini HTTPS'e zorla
    redirect scheme https code 301 if !{ ssl_fc }

# 2. SSL Karşılama ve Yönlendirme (Frontend 443)
frontend https_front
    bind *:443 ssl crt /etc/haproxy/certs/moodle.argeyazilim.tr.pem
    
    # Moodle'a orijinal IP'yi iletmek için X-Forwarded-For header'ı ekleme
    option forwardfor
    http-request set-header X-Forwarded-Proto https
    
    default_backend moodle_cluster

# 3. Moodle Sunucuları (Backend)
backend moodle_cluster
    # Yük dağıtım algoritması (roundrobin: eşit dağıtır, leastconn: en az yoğun olana gönderir)
    balance leastconn
    
    # Moodle'ın HAProxy arkasında olduğunu anlaması için header
    http-request set-header X-Client-IP %[src]

    # Sağlık Kontrolü (Sunucuların ayakta olup olmadığını kontrol eder)
    option httpchk GET /login/index.php
    http-check expect status 200

    # 10 Adet Moodle Sunucumuz (IP'leri kendi ağınıza göre değiştirin)
    server moodle-node-01 192.168.1.11:80 check inter 2000 fall 3 rise 2
    server moodle-node-02 192.168.1.12:80 check inter 2000 fall 3 rise 2
    server moodle-node-03 192.168.1.13:80 check inter 2000 fall 3 rise 2
    server moodle-node-04 192.168.1.14:80 check inter 2000 fall 3 rise 2
    server moodle-node-05 192.168.1.15:80 check inter 2000 fall 3 rise 2
    server moodle-node-06 192.168.1.16:80 check inter 2000 fall 3 rise 2
    server moodle-node-07 192.168.1.17:80 check inter 2000 fall 3 rise 2
    server moodle-node-08 192.168.1.18:80 check inter 2000 fall 3 rise 2
    server moodle-node-09 192.168.1.19:80 check inter 2000 fall 3 rise 2
    server moodle-node-10 192.168.1.20:80 check inter 2000 fall 3 rise 2

# 4. HAProxy İstatistik Paneli (Opsiyonel ama Önerilir)
listen stats
    bind *:8404
    stats enable
    stats uri /haproxy_stats
    stats refresh 10s
    stats auth admin:GUCLU_BIR_SIFRE_YAZIN
🚀 4. HAProxy'i Başlatma ve Doğrulama
Yapılandırma dosyasında sözdizimi (syntax) hatası olup olmadığını kontrol edin:
Bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
Eğer Configuration file is valid çıktısını alıyorsanız, servisi yeniden başlatın:
Bash
sudo systemctl restart haproxy
sudo systemctl enable haproxy
📊 5. Sistemi Test Etme
Artık tarayıcınızdan https://moodle.argeyazilim.tr adresine girdiğinizde istek önce HAProxy'e gelecek, SSL çözülecek ve uygun olan en rölantideki Moodle sunucusuna (leastconn algoritması ile) iletilecektir.
Moodle sunucularınızın anlık yük durumunu ve hangilerinin ayakta olduğunu görmek için tarayıcınızdan istatistik paneline erişebilirsiniz:
http://HAPROXY_IP_ADRESI:8404/haproxy_stats


## 📌 Bölüm 3: Scalelite ve BigBlueButton Kümesi (Cluster) Yapılandırması

Moodle üzerinde açılan canlı derslerin tek bir sunucuya yüklenmesini engellemek ve binlerce kullanıcıya eşzamanlı hizmet verebilmek için **Scalelite** yük dengeleyicisi kullanılmıştır. Scalelite, Moodle'dan gelen toplantı isteklerini karşılar ve havuzdaki en uygun (en az yüke sahip) BigBlueButton (BBB) sunucusuna yönlendirir.

### 🏗️ 1. Scalelite İçin NFS (Kayıt Ortak Alanı) Ayarı

Scalelite'ın en önemli görevlerinden biri, farklı BBB sunucularında yapılan ders kayıtlarını (Recordings) tek bir merkezde toplayıp Moodle'a sunmaktır. Bu nedenle Moodle için kurduğumuz NFS sunucusunda, Scalelite ve BBB sunucuları için ortak bir alan oluşturmalıyız.

Scalelite sunucusunda NFS'i bağlamak için:
```bash
sudo mkdir -p /var/bigbluebutton/spool
sudo mkdir -p /var/bigbluebutton/published
sudo mkdir -p /var/bigbluebutton/unpublished

# NFS Sunucusunu Bağlama (Örn IP: 192.168.1.50)
sudo mount 192.168.1.50:/export/bbb_spool /var/bigbluebutton/spool
sudo mount 192.168.1.50:/export/bbb_published /var/bigbluebutton/published
sudo mount 192.168.1.50:/export/bbb_unpublished /var/bigbluebutton/unpublished
(Not: Bu klasörlerin BBB sunucularına da benzer şekilde NFS üzerinden bağlanması gerekmektedir.)
🛠️ 2. Scalelite Ortam Değişkenleri (.env)
Scalelite sunucusunda (genellikle /var/www/scalelite dizini altında) .env dosyasını yapılandırarak veritabanı ve Redis bağlantılarını tanımlıyoruz:
Kod snippet'i
# URL ve Etiket Tanımlamaları
URL_HOST=scalelite.argeyazilim.tr
SCALELITE_TAG=scalelite.argeyazilim.tr

# Güvenlik Anahtarları
# SECRET_KEY_BASE: 'openssl rand -hex 64' komutu ile üretilmiştir
SECRET_KEY_BASE=BURAYA_URETILEN_64_KARAKTERLIK_ANAHTAR
# LOADBALANCER_SECRET: Moodle'ın Scalelite'a bağlanırken kullanacağı şifre
LOADBALANCER_SECRET=BURAYA_MOODLE_ICIN_OZEL_ANAHTAR

# PostgreSQL ve Redis Bağlantıları (Ortak Sunucular)
DATABASE_URL=postgres://scalelite:SIFRE@POSTGRES_IP_ADRESI/scalelite_production
REDIS_URL=redis://REDIS_IP_ADRESI:6379/0

RAILS_ENV=production
🔄 3. Poller Servisinin Döngüye Alınması (Sürekli Kontrol)
Scalelite'ın, havuza eklediğimiz BBB sunucularının durumunu (çökme, aşırı yüklenme vb.) anlık takip edebilmesi için scalelite-poller servisini sonsuz bir döngüde çalıştırması gerekir.
Bash
sudo nano /etc/systemd/system/scalelite-poller.service
Ini, TOML
[Unit]
Description=Scalelite Poller Loop
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/var/www/scalelite
EnvironmentFile=/var/www/scalelite/.env

# Poller görevini 60 saniyede bir çalıştıracak döngü
ExecStart=/bin/bash -c 'while true; do /root/.rbenv/shims/bundle exec rake poll:all; sleep 60; done'

Restart=always

[Install]
WantedBy=multi-user.target
Servisi aktifleştirin:
Bash
sudo systemctl daemon-reload
sudo systemctl enable scalelite-poller
sudo systemctl start scalelite-poller
➕ 4. BigBlueButton Sunucularını Havuza Dahil Etme
BBB sunucularınızdan (Node) bbb-conf --secret komutu ile aldığınız URL ve Secret bilgilerini kullanarak sunucuları Scalelite havuzuna ekleyin. Bu işlem Scalelite sunucusunda terminal üzerinden yapılır:
Bash
# 1. BBB Sunucusunu Ekleme
RAILS_ENV=production bundle exec rake servers:add[[https://bbb-node-01.argeyazilim.tr/bigbluebutton/api,SUNUCU_1_SECRET_KODU](https://bbb-node-01.argeyazilim.tr/bigbluebutton/api,SUNUCU_1_SECRET_KODU)]
RAILS_ENV=production bundle exec rake servers:add[[https://bbb-node-02.argeyazilim.tr/bigbluebutton/api,SUNUCU_2_SECRET_KODU](https://bbb-node-02.argeyazilim.tr/bigbluebutton/api,SUNUCU_2_SECRET_KODU)]

# 2. Eklenen Sunucuların ID'lerini Listeleme
RAILS_ENV=production bundle exec rake servers

# 3. Sunucuları Aktifleştirme (Listeden alınan ID'ler ile)
RAILS_ENV=production bundle exec rake servers:enable[SUNUCU_1_ID]
RAILS_ENV=production bundle exec rake servers:enable[SUNUCU_2_ID]
🔗 5. Moodle ve Scalelite Entegrasyonu
Altyapı tamamen hazır. Şimdi orkestranın parçalarını birleştiriyoruz.
Herhangi bir Moodle sunucusuna yönetici girişi yapın (veritabanı ortak olduğu için ayar tüm kümeye uygulanacaktır):
Site Yönetimi -> Eklentiler -> BigBlueButton yolunu izleyin.
Ayarları aşağıdaki gibi güncelleyin:
BigBlueButton Server URL: https://scalelite.argeyazilim.tr/bigbluebutton/api (Sonundaki /api kısmı zorunludur).
Shared Secret: Scalelite .env dosyasında belirlediğiniz LOADBALANCER_SECRET değeri.
Ayarları kaydettiğinizde Moodle doğrudan Scalelite ile konuşmaya başlayacaktır. Moodle üzerinden açılan her canlı ders isteği, Scalelite tarafından analiz edilecek ve havuzdaki en uygun BBB sunucusunda başlatılacaktır.
