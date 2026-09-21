# Deployment Ubuntu

Repositori ini berisi Moodle 5.2.3 dan menggunakan `public` sebagai root web. Moodle memerlukan PHP 8.3 atau yang lebih baru. Perintah di bawah ditujukan untuk Ubuntu 24.04. Pengguna Ubuntu 22.04 harus menambahkan repositori PHP yang dijelaskan di bawah atau melakukan upgrade ke Ubuntu 24.04.

Setup ini mengarahkan `https://rainar.net` ke Moodle di `https://elearning.rainar.net`. VM menggunakan `172.20.3.35` pada Bridged Adapter agar dapat diakses perangkat lain di LAN `172.20.3.0/24`. Alamat `10.10.10.35` tetap digunakan pada Host-Only Adapter dan hanya untuk akses dari komputer host. Jika jaringan berubah, sesuaikan alamat bridged, gateway, subnet DNS, sertifikat, dan aturan firewall secara bersamaan.

## Jaringan VirtualBox dan Netplan

Atur adapter VM di VirtualBox sebagai berikut:

- **Adapter 1**: `NAT`, untuk akses internet VM.
- **Adapter 2**: `Bridged Adapter`, pilih `Realtek USB FE Family Controller`, agar VM dapat diakses perangkat lain pada jaringan yang sama.
- **Adapter 3**: `Host-only Adapter`, untuk akses khusus dari komputer host.
- Aktifkan **Cable Connected** pada ketiga adapter.

Gunakan `/etc/netplan/50-cloud-init.yaml` berikut. Pada VM ini `enp0s3` adalah NAT, `enp0s8` adalah bridged, dan `enp0s9` adalah host-only:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 172.20.3.35/24
      optional: true
    enp0s9:
      dhcp4: false
      addresses:
        - 10.10.10.35/24
      optional: true
```

Default route tidak perlu ditambahkan pada `enp0s8` atau `enp0s9` karena koneksi internet tetap menggunakan NAT `enp0s3`. Terapkan dan periksa konfigurasi:

```bash
sudo netplan generate
sudo netplan try
sudo netplan apply
ip -br -4 addr
ip route
ping -c 4 172.20.3.1
```

Hasil akhirnya harus menunjukkan `172.20.3.35/24` pada `enp0s8` dan `10.10.10.35/24` pada `enp0s9`. Perangkat lain harus memakai `172.20.3.35`, bukan alamat host-only `10.10.10.35`. Pastikan `172.20.3.35` belum dipakai perangkat lain atau buat reservasi alamat tersebut pada DHCP server jaringan.

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
ns1     IN A 172.20.3.35
@       IN A 172.20.3.35
elearning IN A 172.20.3.35
```

Pada blok `options` yang sudah ada, batasi DNS ke jaringan LAN:

```conf
listen-on { 127.0.0.1; 172.20.3.35; 10.10.10.35; };
listen-on-v6 { none; };
allow-query { localhost; 172.20.3.0/24; 10.10.10.0/24; };
allow-recursion { localhost; 172.20.3.0/24; 10.10.10.0/24; };
```

```bash
sudo named-checkconf
sudo named-checkzone rainar.net /etc/bind/db.rainar.net
sudo systemctl enable --now named.service
```

Distribusikan `172.20.3.35` sebagai DNS melalui DHCP atau konfigurasikan klien secara manual. Uji dengan `nslookup rainar.net 172.20.3.35` dan `nslookup elearning.rainar.net 172.20.3.35`. Untuk pengujian cepat tanpa mengganti DNS klien, tambahkan `172.20.3.35 rainar.net elearning.rainar.net` ke file `hosts` klien. Jika DNS di-host oleh registrar domain atau penyedia lain, buat record A yang setara di sana dan jangan gunakan zona BIND ini.

## HTTPS Self-signed

```bash
sudo install -d -m 0755 /etc/ssl/localcerts
sudo openssl req -x509 -nodes -newkey rsa:4096 -sha256 -days 730 \
  -keyout /etc/ssl/private/rainar.net.key \
  -out /etc/ssl/localcerts/rainar.net.crt \
  -subj "/CN=rainar.net" \
  -addext "subjectAltName=DNS:rainar.net,DNS:elearning.rainar.net,IP:172.20.3.35,IP:10.10.10.35"
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
sudo -u www-data php /var/www/moodle/admin/cli/install.php \
  --lang=id --wwwroot=https://elearning.rainar.net --dataroot=/var/lib/moodledata \
  --dbtype=mariadb --dbhost=localhost --dbname=moodle --dbuser=moodle \
  --dbpass='CHANGE_ME' --fullname='SMKN 1 CIBINONG' --shortname='SMKN 1 CIBINONG' \
  --adminuser=admin --adminpass='CHANGE_ADMIN_PASSWORD' \
  --adminemail='admin@example.invalid' --non-interactive --agree-license
sudo chown root:www-data /var/www/moodle/config.php
sudo chmod 0640 /var/www/moodle/config.php
```

Nilai `--shortname` menentukan nama situs yang tampil di sisi kiri navbar. Jika Moodle sudah terinstal dengan nama lama, ubah nama situs tanpa instalasi ulang menggunakan:

```bash
sudo -u www-data php /var/www/moodle/admin/cli/cfg.php \
  --component=core --name=fullname --set='SMKN 1 CIBINONG'
sudo -u www-data php /var/www/moodle/admin/cli/cfg.php \
  --component=core --name=shortname --set='SMKN 1 CIBINONG'
sudo -u www-data php /var/www/moodle/admin/cli/purge_caches.php
```

## Fitur Absensi

Repositori sudah menyertakan plugin resmi [Attendance](https://moodle.org/plugins/mod_attendance) di `public/mod/attendance`. Versi plugin yang disertakan mendukung Moodle 5.1 sampai 5.2. Setelah instalasi baru atau setiap kali kode plugin diperbarui, pasang atau perbarui tabel databasenya dengan:

```bash
sudo -u www-data php /var/www/moodle/admin/cli/upgrade.php --non-interactive
sudo -u www-data php /var/www/moodle/admin/cli/purge_caches.php
```

Untuk membuat absensi dalam sebuah kursus:

1. Masuk ke kursus sebagai administrator atau pengajar dan aktifkan **Mode edit**.
2. Klik **Tambahkan aktivitas atau sumber daya**, lalu pilih **Attendance**.
3. Isi nama aktivitas, misalnya `Absensi`, kemudian simpan.
4. Buka aktivitas tersebut dan pilih **Add session** untuk membuat jadwal pertemuan.
5. Gunakan status bawaan `Present`, `Absent`, `Late`, dan `Excused`, atau sesuaikan melalui tab **Status set**.
6. Aktifkan **Allow students to record own attendance** pada sesi jika siswa diperbolehkan mengisi sendiri. Gunakan password atau QR code untuk membatasi akses.
7. Gunakan tab **Report** atau **Export** untuk melihat dan mengunduh rekap kehadiran.

Jika plugin belum muncul dalam pemilih aktivitas, periksa bahwa direktori `/var/www/moodle/public/mod/attendance` tersedia dan pastikan perintah upgrade di atas selesai tanpa galat.

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

### Tampilkan CSV sebagai Category 1 di File Picker

Folder **Category 1** yang sudah terlihat di dalam **Content bank** bukan folder umum untuk CSV. Content bank digunakan untuk konten seperti H5P, sehingga `users.csv` tidak boleh dimasukkan ke folder tersebut. Agar **Category 1** muncul sebagai pilihan tersendiri di panel kiri file picker dan berisi `users.csv`, buat instance **File system repository** dengan nama yang sama:

1. Masuk sebagai administrator Moodle.
2. Buka **Administrasi situs > Plugin > Repositori > Kelola repositori**.
3. Cari **File system**, lalu ubah statusnya menjadi **Aktif dan terlihat** (`Enabled and visible`).
4. Klik **Buat instance repositori** (`Create a repository instance`).
5. Isi nama instance dengan `Category 1`.
6. Pilih direktori `import-users`, lalu simpan. Opsi relative files tidak perlu diaktifkan.
7. Buka **Administrasi situs > Pengguna > Akun > Upload pengguna**.
8. Klik **Choose a file...**. Pilihan **Category 1** sekarang muncul langsung di panel kiri file picker.
9. Klik **Category 1**, pilih `users.csv`, lalu klik **Select this file**.
10. Gunakan separator koma dan encoding UTF-8, periksa preview, lalu jalankan import.

Dengan konfigurasi ini akan ada dua nama **Category 1** yang berbeda:

- **Content bank > Category 1** adalah area konten Moodle dan tidak digunakan untuk CSV pengguna.
- **Category 1** di panel kiri adalah File system repository yang membaca `/var/lib/moodledata/repository/import-users/users.csv`.

Moodle hanya menawarkan direktori yang berada di `/var/lib/moodledata/repository/` saat membuat File system repository. Jika `import-users` tidak muncul atau **Category 1** belum terlihat di file picker, jalankan:

```bash
sudo chown -R www-data:www-data /var/lib/moodledata/repository
sudo chmod 0770 /var/lib/moodledata/repository
sudo chmod 0770 /var/lib/moodledata/repository/import-users
sudo -u www-data php /var/www/moodle/admin/cli/purge_caches.php
```

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
sudo sh -c 'printf "%s\n" "* * * * * www-data /usr/bin/php /var/www/moodle/admin/cli/cron.php >/dev/null 2>&1" > /etc/cron.d/moodle'
sudo chmod 0644 /etc/cron.d/moodle
sudo ufw allow OpenSSH
sudo ufw allow 'Apache Full'
sudo ufw allow from 172.20.3.0/24 to any port 53
sudo ufw allow from 10.10.10.0/24 to any port 53
sudo ufw enable
systemctl --no-pager --full status apache2 mariadb named.service
curl -kI https://rainar.net/
curl -kI https://elearning.rainar.net/
```

Buat cadangan database dan `/var/lib/moodledata` sebelum upgrade. Jangan pernah commit database, direktori tersebut, private key TLS, atau `config.php` ke Git.
