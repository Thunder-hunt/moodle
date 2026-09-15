# Ubuntu 24.04 Deployment

This repository contains Moodle 5.2.3 and uses `public` as the web root. Moodle requires PHP 8.3 or newer; this guide installs PHP 8.3 on Ubuntu 24.04.

This setup serves the Apache default site at `https://rainar.net` and Moodle at `https://elearning.rainar.net`. Replace `192.168.1.10` with the VM's static IP and `192.168.1.0/24` with your LAN subnet. The DNS zone can be publicly hosted or managed by BIND9 for LAN-only access.

## Install packages

```bash
sudo apt update
sudo apt install -y apache2 bind9 bind9-utils mariadb-server git openssl \
  php8.3 php8.3-cli libapache2-mod-php8.3 php8.3-mysql php8.3-curl \
  php8.3-gd php8.3-intl php8.3-mbstring php8.3-soap php8.3-xml \
  php8.3-xmlrpc php8.3-zip php8.3-bcmath
```

The Sodium extension is included with supported PHP versions on Ubuntu 24.04, so it does not need a separate package.

## Clone and protect Moodle

```bash
sudo git clone https://github.com/Thunder-hunt/moodle.git /var/www/moodle
sudo mkdir -p /var/lib/moodledata
sudo chown -R root:www-data /var/www/moodle
sudo find /var/www/moodle -type d -exec chmod 0755 {} \;
sudo find /var/www/moodle -type f -exec chmod 0644 {} \;
sudo chown -R www-data:www-data /var/lib/moodledata
sudo chmod 0770 /var/lib/moodledata
```

Keep `moodledata` outside the web root. It and `config.php` are ignored by Git.

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

For MariaDB 10.6+, use `/etc/mysql/mariadb.conf.d/60-moodle.cnf`:

```ini
[mysqld]
innodb_file_per_table = 1
max_allowed_packet = 256M
```

```bash
sudo systemctl restart mariadb
```

## BIND9 DNS

If BIND9 is authoritative for `rainar.net`, add this zone to `/etc/bind/named.conf.local`:

```conf
zone "rainar.net" {
    type primary;
    file "/etc/bind/db.rainar.net";
};
```

Create `/etc/bind/db.rainar.net`:

```dns
$TTL 86400
@       IN SOA ns1.rainar.net. admin.rainar.net. (2026091501 3600 1800 604800 86400)
        IN NS ns1.rainar.net.
ns1     IN A 192.168.1.10
@       IN A 192.168.1.10
elearning IN A 192.168.1.10
```

In the existing `options` block, restrict DNS to the LAN:

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

Advertise `192.168.1.10` as DNS through DHCP or configure clients manually. Test with `nslookup rainar.net 192.168.1.10` and `nslookup elearning.rainar.net 192.168.1.10`. If DNS is hosted by your domain registrar or another provider, create equivalent A records there instead of using this BIND zone.

## Self-signed HTTPS

```bash
sudo install -d -m 0755 /etc/ssl/localcerts
sudo openssl req -x509 -nodes -newkey rsa:4096 -sha256 -days 730 \
  -keyout /etc/ssl/private/rainar.net.key \
  -out /etc/ssl/localcerts/rainar.net.crt \
  -subj "/CN=rainar.net" \
  -addext "subjectAltName=DNS:rainar.net,DNS:elearning.rainar.net,IP:192.168.1.10"
sudo chmod 0600 /etc/ssl/private/rainar.net.key
```

Import the `.crt` into each client device's trusted root store to remove browser warnings.

## Apache

```bash
sudo a2enmod ssl rewrite headers expires
sudoedit /etc/apache2/sites-available/moodle.conf
```

```apache
<VirtualHost *:80>
    ServerName rainar.net
    ServerAlias www.rainar.net
    DocumentRoot /var/www/html
    Redirect permanent / https://rainar.net/
</VirtualHost>
<VirtualHost *:443>
    ServerName rainar.net
    ServerAlias www.rainar.net
    DocumentRoot /var/www/html
    SSLEngine on
    SSLCertificateFile /etc/ssl/localcerts/rainar.net.crt
    SSLCertificateKeyFile /etc/ssl/private/rainar.net.key
    <Directory /var/www/html>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
        DirectoryIndex index.html
    </Directory>
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

## PHP and Moodle installation

Set these values in the active Apache PHP configuration file, typically `/etc/php/<version>/apache2/php.ini`, then restart Apache:

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
  --lang=en --wwwroot=https://elearning.rainar.net --dataroot=/var/lib/moodledata \
  --dbtype=mariadb --dbhost=localhost --dbname=moodle --dbuser=moodle \
  --dbpass='CHANGE_ME' --fullname='My Moodle' --shortname='Moodle' \
  --adminuser=admin --adminpass='CHANGE_ADMIN_PASSWORD' \
  --adminemail='admin@example.invalid' --non-interactive --agree-license
sudo chown root:www-data /var/www/moodle/config.php
sudo chmod 0640 /var/www/moodle/config.php
```

## Cron, firewall, and checks

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

Back up the database and `/var/lib/moodledata` before upgrades. Never commit either, the TLS private key, or `config.php`.
