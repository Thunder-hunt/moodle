# Deployment Ubuntu

Repositori ini berisi Moodle 5.2.3 dan menggunakan `public` sebagai root web. Moodle memerlukan PHP 8.3 atau yang lebih baru. Perintah di bawah ditujukan untuk Ubuntu 24.04. Pengguna Ubuntu 22.04 harus menambahkan repositori PHP yang dijelaskan di bawah atau melakukan upgrade ke Ubuntu 24.04.

Setup ini mengarahkan `https://rainar.net` ke Moodle di `https://elearning.rainar.net`. Ganti `192.168.1.10` dengan IP statis VM dan `192.168.1.0/24` dengan subnet LAN Anda. Zona DNS dapat di-host secara publik atau dikelola oleh BIND9 untuk akses khusus LAN.

## Instal Paket

Di Ubuntu 24.04, instal PHP 8.3 dari repositori standar Ubuntu:

```bash
sudo apt update
sudo apt install -y apache2 bind9 bind9-utils mariadb-server git openssl \
  php8.3 php8.3-cli libapache2-mod-php8.3 php8.3-mysql php8.3-curl \
  php8.3-gd php8.3-intl php8.3-mbstring php8.3-soap php8.3-xml \
  php8.3-xmlrpc php8.3-zip php8.3-bcmath
```

Jika server menggunakan Ubuntu 22.04 (Jammy), repositori standar menyediakan PHP 8.1 yang terlalu lama untuk versi Moodle ini. Tambahkan repositori PHP Ondrej terlebih dahulu, kemudian jalankan perintah instalasi paket di atas:

```bash
sudo apt install -y software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt update
```

Periksa versi yang terinstal sebelum melanjutkan:

```bash
php -v
```

Output harus menunjukkan PHP 8.3 atau yang lebih baru.

Ekstensi Sodium sudah termasuk dalam versi PHP yang didukung pada Ubuntu 24.04, sehingga tidak memerlukan paket terpisah.

## Clone dan Amankan Moodle

```bash
sudo git clone https://github.com/Thunder-hunt/moodle.git /var/www/moodle
sudo mkdir -p /var/lib/moodledata/repository/import-users
sudo chown -R root:www-data /var/www/moodle
sudo find /var/www/moodle -type d -exec chmod 0755 {} \;
sudo find /var/www/moodle -type f -exec chmod 0644 {} \;
sudo chown -R www-data:www-data /var/lib/moodledata
sudo chmod 0770 /var/lib/moodledata
sudo chmod 0770 /var/lib/moodledata/repository /var/lib/moodledata/repository/import-users
sudo install -o www-data -g www-data -m 0640 \
  /var/www/moodle/users.csv /var/lib/moodledata/repository/import-users/users.csv
```

Simpan `moodledata` di luar root web. Direktori tersebut dan `config.php` diabaikan oleh Git. `users.csv` ikut ter-clone di root repositori dan disalin saat setup ke `repository/import-users`, agar bisa dipilih melalui file picker Moodle. Jika mengubah CSV setelah deployment, salin ulang dengan perintah `sudo install` di atas.

## MariaDB

```bash
sudo mariadb-secure-installation
sudo mariadb
```

```sql
CREATE DATABASE moodle DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodle'@'localhost' IDENTIFIED BY 'CHANGE_ME';
GRANT ALL PRIVILEGES ON moodle.* TO 'moodle'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Untuk MariaDB 10.6 atau yang lebih baru, gunakan `/etc/mysql/mariadb.conf.d/60-moodle.cnf`:

```ini
[mysqld]
innodb_file_per_table = 1
max_allowed_packet = 256M
```

```bash
sudo systemctl restart mariadb
```

## DNS BIND9

Jika BIND9 menjadi server authoritative untuk `rainar.net`, tambahkan zona ini ke `/etc/bind/named.conf.local`:

```conf
zone "rainar.net" {
    type primary;
    file "/etc/bind/db.rainar.net";
};
```

Buat `/etc/bind/db.rainar.net`:

```dns
$TTL 86400
@       IN SOA ns1.rainar.net. admin.rainar.net. (2026091501 3600 1800 604800 86400)
        IN NS ns1.rainar.net.
ns1     IN A 192.168.1.10
@       IN A 192.168.1.10
elearning IN A 192.168.1.10
```

Pada blok `options` yang sudah ada, batasi DNS ke jaringan LAN:

```conf
listen-on { 127.0.0.1; 192.168.1.10; };
listen-on-v6 { none; };
allow-query { localhost; 192.168.1.0/24; };
allow-recursion { localhost; 192.168.1.0/24; };
```

```bash
sudo named-checkconf
sudo named-checkzone rainar.net /etc/bind/db.rainar.net
sudo systemctl enable --now named.service
```

Distribusikan `192.168.1.10` sebagai DNS melalui DHCP atau konfigurasikan klien secara manual. Uji dengan `nslookup rainar.net 192.168.1.10` dan `nslookup elearning.rainar.net 192.168.1.10`. Jika DNS di-host oleh registrar domain atau penyedia lain, buat record A yang setara di sana dan jangan gunakan zona BIND ini.

## HTTPS Self-signed

```bash
sudo install -d -m 0755 /etc/ssl/localcerts
sudo openssl req -x509 -nodes -newkey rsa:4096 -sha256 -days 730 \
  -keyout /etc/ssl/private/rainar.net.key \
  -out /etc/ssl/localcerts/rainar.net.crt \
  -subj "/CN=rainar.net" \
  -addext "subjectAltName=DNS:rainar.net,DNS:elearning.rainar.net,IP:192.168.1.10"
sudo chmod 0600 /etc/ssl/private/rainar.net.key
```

Impor file `.crt` ke penyimpanan root tepercaya pada setiap perangkat klien untuk menghilangkan peringatan browser.

## Apache

```bash
sudo a2enmod ssl rewrite headers expires
sudoedit /etc/apache2/sites-available/moodle.conf
```

```apache
<VirtualHost *:80>
    ServerName rainar.net
    ServerAlias www.rainar.net
    Redirect permanent / https://elearning.rainar.net/
</VirtualHost>
<VirtualHost *:443>
    ServerName rainar.net
    ServerAlias www.rainar.net
    SSLEngine on
    SSLCertificateFile /etc/ssl/localcerts/rainar.net.crt
    SSLCertificateKeyFile /etc/ssl/private/rainar.net.key
    Redirect permanent / https://elearning.rainar.net/
</VirtualHost>

<VirtualHost *:80>
    ServerName elearning.rainar.net
    Redirect permanent / https://elearning.rainar.net/
</VirtualHost>

<VirtualHost *:443>
    ServerName elearning.rainar.net
    DocumentRoot /var/www/moodle/public
    SSLEngine on
    SSLCertificateFile /etc/ssl/localcerts/rainar.net.crt
    SSLCertificateKeyFile /etc/ssl/private/rainar.net.key
    <Directory /var/www/moodle/public>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
        DirectoryIndex index.php
        RewriteEngine On
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteRule ^(.+)$ /r.php?file=/$1 [L,QSA]
    </Directory>
    <Directory /var/www/moodle>
        Require all denied
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/moodle-error.log
    CustomLog ${APACHE_LOG_DIR}/moodle-access.log combined
</VirtualHost>
```

```bash
sudo a2ensite moodle.conf
sudo a2dissite 000-default.conf default-ssl.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

## Instalasi PHP dan Moodle

Atur nilai berikut pada file konfigurasi PHP Apache yang aktif, biasanya `/etc/php/<version>/apache2/php.ini`, lalu mulai ulang Apache:

```ini
memory_limit = 256M
post_max_size = 100M
upload_max_filesize = 100M
max_execution_time = 300
max_input_vars = 5000
```

```bash
sudo systemctl restart apache2
sudo -u www-data php /var/www/moodle/public/admin/cli/install.php \
  --lang=id --wwwroot=https://elearning.rainar.net --dataroot=/var/lib/moodledata \
  --dbtype=mariadb --dbhost=localhost --dbname=moodle --dbuser=moodle \
  --dbpass='CHANGE_ME' --fullname='My Moodle' --shortname='Moodle' \
  --adminuser=admin --adminpass='CHANGE_ADMIN_PASSWORD' \
  --adminemail='admin@example.invalid' --non-interactive --agree-license
sudo chown root:www-data /var/www/moodle/config.php
sudo chmod 0640 /var/www/moodle/config.php
```

## Import Pengguna dari CSV di Server

Untuk latihan kelas ini, `users.csv` di root repositori memang berisi akun contoh dengan password yang sama dan ikut di-clone. Jangan gunakan akun atau password ini untuk pengguna nyata; siapa pun yang punya akses ke repositori dapat membacanya. Root web adalah `/var/www/moodle/public`, jadi jangan pindahkan CSV ke dalam `public/`. Moodle membaca salinan CSV dari:

```text
/var/lib/moodledata/repository/import-users/
```

CSV untuk membuat pengguna baru minimal berisi `username`, `password`, `firstname`, `lastname`, dan `email`. File `users.csv` yang ikut di-clone sudah menggunakan format ini. Contoh:

```csv
username,password,firstname,lastname,email
user1,User1234!,User,1,user1@example.com
user2,User1234!,User,2,user2@example.com
```

Password latihan seluruh akun adalah `User1234!` dan sudah memenuhi kebijakan password standar Moodle. Alamat `example.com` hanya untuk latihan, bukan email pengguna nyata.

### Pilih CSV melalui Web Moodle

1. Masuk sebagai administrator Moodle.
2. Buka **Administrasi situs > Plugin > Repositori > Kelola repositori**.
3. Aktifkan repositori **File system**, kemudian buat instance repository.
4. Beri nama `Import Users` dan pilih subdirektori `import-users`.
5. Buka **Administrasi situs > Pengguna > Akun > Upload pengguna**.
6. Pada pemilih file, pilih repository **Import Users**, lalu pilih `users.csv`.
7. Gunakan delimiter koma dan encoding UTF-8, periksa preview, lalu jalankan import.

Moodle hanya menampilkan subdirektori yang berada di `/var/lib/moodledata/repository/`. Jika `import-users` tidak muncul, periksa kembali lokasi dan permission direktori, kemudian bersihkan cache Moodle.

### Import Langsung melalui CLI

Administrator server juga dapat menjalankan import tanpa file picker web:

```bash
sudo -u www-data php /var/www/moodle/public/admin/tool/uploaduser/cli/uploaduser.php \
  --file=/var/lib/moodledata/repository/import-users/users.csv
```

Periksa ringkasan hasil import yang dicetak oleh perintah tersebut. Tampilkan seluruh opsi jika perlu mengubah mode pembuatan atau pembaruan pengguna:

```bash
sudo -u www-data php /var/www/moodle/public/admin/tool/uploaduser/cli/uploaduser.php --help
```

Setelah import berhasil, salinan di `moodledata` dapat dihapus jika tidak lagi diperlukan. File asli tetap berada di repositori Git dan akan ikut saat clone berikutnya:

```bash
sudo rm /var/lib/moodledata/repository/import-users/users.csv
```

## Cron, Firewall, dan Pemeriksaan

```bash
sudo sh -c 'printf "%s\n" "* * * * * www-data /usr/bin/php /var/www/moodle/public/admin/cli/cron.php >/dev/null 2>&1" > /etc/cron.d/moodle'
sudo chmod 0644 /etc/cron.d/moodle
sudo ufw allow OpenSSH
sudo ufw allow 'Apache Full'
sudo ufw allow from 192.168.1.0/24 to any port 53
sudo ufw enable
systemctl --no-pager --full status apache2 mariadb named.service
curl -kI https://rainar.net/
curl -kI https://elearning.rainar.net/
```

Buat cadangan database dan `/var/lib/moodledata` sebelum upgrade. Jangan pernah commit database, direktori tersebut, private key TLS, atau `config.php` ke Git.
