# Ubuntu 24.04 LAN Deployment

This repository contains Moodle 5.2.3 and uses `public` as the web root. Ubuntu will select the default PHP version for Ubuntu 24.04.

Replace `moodle.lan` with your LAN hostname, `192.168.1.10` with the VM's static IP, and `192.168.1.0/24` with your LAN subnet.

## Install packages

```bash
sudo apt update
sudo apt install -y apache2 bind9 bind9-utils mariadb-server git openssl \
  php php-cli libapache2-mod-php php-mysql php-curl \
  php-gd php-intl php-mbstring php-soap php-xml \
  php-xmlrpc php-zip php-bcmath php-sodium
```

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

## BIND9 LAN DNS

Add this zone to `/etc/bind/named.conf.local`:

```conf
zone "lan" {
    type primary;
    file "/etc/bind/db.lan";
};
```

Create `/etc/bind/db.lan`:

```dns
$TTL 86400
@ IN SOA ns1.lan. admin.lan. (2026091501 3600 1800 604800 86400)
  IN NS ns1.lan.
ns1 IN A 192.168.1.10
moodle IN A 192.168.1.10
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
sudo named-checkzone lan /etc/bind/db.lan
sudo systemctl enable --now bind9
```

Advertise `192.168.1.10` as DNS through DHCP or configure clients manually. Test with `nslookup moodle.lan 192.168.1.10`.

## Self-signed HTTPS

```bash
sudo install -d -m 0755 /etc/ssl/localcerts
sudo openssl req -x509 -nodes -newkey rsa:4096 -sha256 -days 730 \
  -keyout /etc/ssl/private/moodle.lan.key \
  -out /etc/ssl/localcerts/moodle.lan.crt \
  -subj "/CN=moodle.lan" \
  -addext "subjectAltName=DNS:moodle.lan,IP:192.168.1.10"
sudo chmod 0600 /etc/ssl/private/moodle.lan.key
```

Import the `.crt` into each client device's trusted root store to remove browser warnings.

## Apache

```bash
sudo a2enmod ssl rewrite headers expires
sudoedit /etc/apache2/sites-available/moodle.conf
```

```apache
<VirtualHost *:80>
    ServerName moodle.lan
    Redirect permanent / https://moodle.lan/
</VirtualHost>
<VirtualHost *:443>
    ServerName moodle.lan
    DocumentRoot /var/www/moodle/public
    SSLEngine on
    SSLCertificateFile /etc/ssl/localcerts/moodle.lan.crt
    SSLCertificateKeyFile /etc/ssl/private/moodle.lan.key
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
  --lang=en --wwwroot=https://moodle.lan --dataroot=/var/lib/moodledata \
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
systemctl --no-pager --full status apache2 mariadb bind9
curl -kI https://moodle.lan/
```

Back up the database and `/var/lib/moodledata` before upgrades. Never commit either, the TLS private key, or `config.php`.
