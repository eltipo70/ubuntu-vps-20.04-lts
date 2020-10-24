# ubuntu-vps-20.04-lts

#### Production Web Server Installation Guide

Following this guide you will be able to install and configure website based on Ubuntu 20.04 LTS 64Bit (ARM64 anor AMD64), NGINX 1.19, TLSv1.3, PHP 7.4, MariaDB 10.5, Redis, UFW and fail2ban. You will achieve an A+ rating from Qualys SSL Labs. We will request and implement your ssl certificate(s) from Let’s Encrypt – you only have to ammend all the ```diff + red``` marked values like `your.domain.io`, `192.168.2.x` with regards to your environment!

## Table of content
1. [Prepare your server.](#section1)
2. [Install NGINX 1.19.](#section2)
3. [PHP 7.4.](#section3)
4. [MariaDB 10.5.](#section4)
5. [Redis Server.](#section5)
6. [Symfony Application (SSL enabled, A+).](#section6)
7. [Hardening (faul2ban & ufw).](#section7)

### <a name="section1"></a> 1. Prepare your server. 

Change into sudo mode:
```
sudo -s
```
Prepare your server for the installation itself:
```
apt install -y \
    curl \
    gnupg2 \
    git \
    lsb-release \
    ssl-cert \
    ca-certificates \
    apt-transport-https \
    tree \
    locate \
    software-properties-common \
    dirmngr \
    screen \
    htop \
    net-tools \
    zip unzip \
    curl \
    ffmpeg \
    ghostscript \
    libfile-fcntllock-perl
```
Add new software repositories:
```
cd /etc/apt/sources.list.d
```
```
echo "deb [arch=amd64] http://nginx.org/packages/mainline/ubuntu $(lsb_release -cs) nginx" | tee nginx.list
```
```
echo "deb [arch=amd64] http://ppa.launchpad.net/ondrej/php/ubuntu $(lsb_release -cs) main" | tee php.list
```
```
echo "deb [arch=amd64] http://ftp.hosteurope.de/mirror/mariadb.org/repo/10.5/ubuntu $(lsb_release -cs) main" | tee mariadb.list
```

Download the required keys to trust all the new sources:
```
curl -fsSL https://nginx.org/keys/nginx_signing.key | sudo apt-key add -
```
```
apt-key adv --recv-keys --keyserver hkps://keyserver.ubuntu.com:443 4F4EA0AAE5267A6C
```
```
apt-key adv --recv-keys --keyserver hkps://keyserver.ubuntu.com:443 0xF1656F24C74CD1D8
```

Update your server and generate self signed certificates:
```
apt update && apt upgrade -y
```
```
make-ssl-cert generate-default-snakeoil -y
```

Remove old nginx software if exists (optional):
```
apt remove nginx nginx-extras nginx-common nginx-full -y --allow-change-held-packages
```

### <a name="section2"></a> 2. Install and configure NGINX.


First ensure Apache(2) isn’t running otherwise NGINX won’t start because the required port (:80) would be in use by Apache(2):
```
systemctl stop apache2.service && systemctl disable apache2.service
```
```
apt install nginx -y
```
```
systemctl enable nginx.service
```

Change the default nginx configuration:
```
mv /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak && touch /etc/nginx/nginx.conf
```
```
nano /etc/nginx/nginx.conf
```
Paste all the following rows to the new file. Substitute the red marked parameters properly:
```
user www-data;
worker_processes auto;
pid /var/run/nginx.pid;
events {
worker_connections 1024;
multi_accept on; use epoll;
}
http {
server_names_hash_bucket_size 64;
upstream php-handler {
server unix:/run/php/php7.4-fpm.sock;
}
access_log /var/log/nginx/access.log;
error_log /var/log/nginx/error.log warn;
set_real_ip_from 127.0.0.1;
set_real_ip_from 192.168.2.0/24;
# please customize the ip-range properly
real_ip_header X-Forwarded-For;
real_ip_recursive on;
include /etc/nginx/mime.types;
#include /etc/nginx/proxy.conf;
#include /etc/nginx/ssl.conf;
#include /etc/nginx/header.conf;
#include /etc/nginx/optimization.conf; 
default_type application/octet-stream;
sendfile on;
send_timeout 3600;
tcp_nopush on;
tcp_nodelay on;
open_file_cache max=500 inactive=10m;
open_file_cache_errors on;
keepalive_timeout 65;
reset_timedout_connection on;
server_tokens off;
resolver 192.168.2.1 valid=30s;
#resolver 127.0.0.53 valid=30s; *is recommended but requires a valid DNS configuration*
resolver_timeout 5s;
include /etc/nginx/conf.d/*.conf;
}
```

Test and restart the webserver
```
nginx -t && service nginx restart
```

Create four folders and apply the appropriate permissions:
```
mkdir -p /var/www/your.domain.io /var/www/letsencrypt
```
```
chown -R www-data:www-data /var/www
```
The preparation and installation of nginx has finished and we will install PHP latest in the next chapter. 

### <a name="section3"></a> 3. Install and configure PHP 7.4 (fpm)

The PHP repository was enabled in chapter 1 – so just perform the following statement to install PHP:
```
apt update && apt install -y \
    php7.4-fpm \
    php7.4-gd \
    php7.4-mysql \
    php7.4-curl \
    php7.4-xml \
    php7.4-zip \
    php7.4-intl \
    php7.4-mbstring \
    php7.4-json \
    php7.4-bz2 \
    php7.4-ldap \
    php-apcu \
    imagemagick \
    php-imagick \
    php-smbclient
```

Awesome, PHP 7.4 is already installed. Verify your timezone settings :
```
date
```

If it is not set properly customize it as examplarily shown for your location (Moscow, Russia in my case):
```
timedatectl set-timezone Europe/Moscow
```

Backup and then tweak PHP for optimization and security reasons:
```
cp /etc/php/7.4/fpm/pool.d/www.conf /etc/php/7.4/fpm/pool.d/www.conf.bak
cp /etc/php/7.4/cli/php.ini /etc/php/7.4/cli/php.ini.bak
cp /etc/php/7.4/fpm/php.ini /etc/php/7.4/fpm/php.ini.bak
cp /etc/php/7.4/fpm/php-fpm.conf /etc/php/7.4/fpm/php-fpm.conf.bak
cp /etc/ImageMagick-6/policy.xml /etc/ImageMagick-6/policy.xml.bak
```
```
sed -i "s/;env\[HOSTNAME\] = /env[HOSTNAME] = /" /etc/php/7.4/fpm/pool.d/www.conf
sed -i "s/;env\[TMP\] = /env[TMP] = /" /etc/php/7.4/fpm/pool.d/www.conf
sed -i "s/;env\[TMPDIR\] = /env[TMPDIR] = /" /etc/php/7.4/fpm/pool.d/www.conf
sed -i "s/;env\[TEMP\] = /env[TEMP] = /" /etc/php/7.4/fpm/pool.d/www.conf
sed -i "s/;env\[PATH\] = /env[PATH] = /" /etc/php/7.4/fpm/pool.d/www.conf
```
```
sed -i "s/output_buffering =.*/output_buffering = 'Off'/" /etc/php/7.4/cli/php.ini
sed -i "s/max_execution_time =.*/max_execution_time = 3600/" /etc/php/7.4/cli/php.ini
sed -i "s/max_input_time =.*/max_input_time = 3600/" /etc/php/7.4/cli/php.ini
sed -i "s/post_max_size =.*/post_max_size = 10240M/" /etc/php/7.4/cli/php.ini
sed -i "s/upload_max_filesize =.*/upload_max_filesize = 10240M/" /etc/php/7.4/cli/php.ini
sed -i "s/;date.timezone.*/date.timezone = Europe\/\Moscow/" /etc/php/7.4/cli/php.ini
```
```
sed -i "s/memory_limit = 128M/memory_limit = 1024M/" /etc/php/7.4/fpm/php.ini
sed -i "s/output_buffering =.*/output_buffering = 'Off'/" /etc/php/7.4/fpm/php.ini
sed -i "s/max_execution_time =.*/max_execution_time = 3600/" /etc/php/7.4/fpm/php.ini
sed -i "s/max_input_time =.*/max_input_time = 3600/" /etc/php/7.4/fpm/php.ini
sed -i "s/post_max_size =.*/post_max_size = 10240M/" /etc/php/7.4/fpm/php.ini
sed -i "s/upload_max_filesize =.*/upload_max_filesize = 10240M/" /etc/php/7.4/fpm/php.ini
sed -i "s/;date.timezone.*/date.timezone = Europe\/\Moscow/" /etc/php/7.4/fpm/php.ini
sed -i "s/;session.cookie_secure.*/session.cookie_secure = True/" /etc/php/7.4/fpm/php.ini
sed -i "s/;opcache.enable=.*/opcache.enable=1/" /etc/php/7.4/fpm/php.ini
sed -i "s/;opcache.enable_cli=.*/opcache.enable_cli=1/" /etc/php/7.4/fpm/php.ini
sed -i "s/;opcache.memory_consumption=.*/opcache.memory_consumption=128/" /etc/php/7.4/fpm/php.ini
sed -i "s/;opcache.interned_strings_buffer=.*/opcache.interned_strings_buffer=8/" /etc/php/7.4/fpm/php.ini
sed -i "s/;opcache.max_accelerated_files=.*/opcache.max_accelerated_files=10000/" /etc/php/7.4/fpm/php.ini
sed -i "s/;opcache.revalidate_freq=.*/opcache.revalidate_freq=1/" /etc/php/7.4/fpm/php.ini
sed -i "s/;opcache.save_comments=.*/opcache.save_comments=1/" /etc/php/7.4/fpm/php.ini
```
```
sed -i '$aapc.enable_cli=1' /etc/php/7.4/mods-available/apcu.ini
```
```
sed -i "s/rights=\"none\" pattern=\"PS\"/rights=\"read|write\" pattern=\"PS\"/" /etc/ImageMagick-6/policy.xml
sed -i "s/rights=\"none\" pattern=\"EPI\"/rights=\"read|write\" pattern=\"EPI\"/" /etc/ImageMagick-6/policy.xml
sed -i "s/rights=\"none\" pattern=\"PDF\"/rights=\"read|write\" pattern=\"PDF\"/" /etc/ImageMagick-6/policy.xml
sed -i "s/rights=\"none\" pattern=\"XPS\"/rights=\"read|write\" pattern=\"XPS\"/" /etc/ImageMagick-6/policy.xml
```

Restart both services, php and nginx:
```
service php7.4-fpm restart
```
```
service nginx restart
```
PHP was successfully installed and configured – we will go ahead with the installation of MariaDB.

### <a name="section4"></a> 4. Install, harden and configure MariaDB 10.5

Install MariaDB by issuing the following statement:
```
apt update && apt install mariadb-server -y
```

Verify your database server version: 
```
mysql --version
```

#### Harden and secure your MariaDB by issuing:
```
mysql_secure_installation
```
```
Switch to unix_socket authentication [Y/n] N
```
```
Enter current password for root (enter for none): <ENTER> or type the password
Set root password? [Y/n] Y
```

If already set during the MariaDB installation you will be asked wether to change or keep the password
```
Remove anonymous users? [Y/n] Y
Disallow root login remotely? [Y/n] Y
Remove test database and access to it? [Y/n] Y
Reload privilege tables now? [Y/n] Y
```

With regards to your website we optimize the configuration of MariaDB:
```
service mysql stop
```
```
mv /etc/mysql/my.cnf /etc/mysql/my.cnf.bak
```
```
nano /etc/mysql/my.cnf
```

Paste all the following rows: 
```
[client]
 default-character-set = utf8mb4
 port = 3306
 socket = /var/run/mysqld/mysqld.sock
[mysqld_safe]
 log_error = /var/log/mysql/mysql_error.log
 nice = 0
 socket = /var/run/mysqld/mysqld.sock
[mysqld]
 basedir = /usr
 bind-address = 127.0.0.1
 binlog_format = ROW
 bulk_insert_buffer_size = 16M
 character-set-server = utf8mb4
 collation-server = utf8mb4_general_ci
 concurrent_insert = 2
 connect_timeout = 5
 datadir = /var/lib/mysql
 default_storage_engine = InnoDB
 expire_logs_days = 7
 general_log_file = /var/log/mysql/mysql.log
 general_log = 0
 innodb_buffer_pool_size = 1024M
 innodb_buffer_pool_instances = 1
 innodb_flush_log_at_trx_commit = 2
 innodb_log_buffer_size = 32M
 innodb_max_dirty_pages_pct = 90
 innodb_file_per_table = 1
 innodb_open_files = 400
 innodb_io_capacity = 4000
 innodb_flush_method = O_DIRECT
 key_buffer_size = 128M
 lc_messages_dir = /usr/share/mysql
 lc_messages = en_US
 log_bin = /var/log/mysql/mariadb-bin
 log_bin_index = /var/log/mysql/mariadb-bin.index
 log_error = /var/log/mysql/mysql_error.log
 log_slow_verbosity = query_plan
 log_warnings = 2
 long_query_time = 1
 max_allowed_packet = 16M
 max_binlog_size = 100M
 max_connections = 200
 max_heap_table_size = 64M
 myisam_recover_options = BACKUP
 myisam_sort_buffer_size = 512M
 port = 3306
 pid-file = /var/run/mysqld/mysqld.pid
 query_cache_limit = 2M
 query_cache_size = 64M
 query_cache_type = 1
 query_cache_min_res_unit = 2k
 read_buffer_size = 2M
 read_rnd_buffer_size = 1M
 skip-external-locking
 skip-name-resolve
 slow_query_log_file = /var/log/mysql/mariadb-slow.log
 slow-query-log = 1
 socket = /var/run/mysqld/mysqld.sock
 sort_buffer_size = 4M
 table_open_cache = 400
 thread_cache_size = 128
 tmp_table_size = 64M
 tmpdir = /tmp
 transaction_isolation = READ-COMMITTED
 user = mysql
 wait_timeout = 600
[mysqldump]
 max_allowed_packet = 16M
 quick
 quote-names
[isamchk]
 key_buffer = 16M 
```

Restart MariaDB and connect to MariaDB:
```
service mysql restart
mysql -uroot -p
```

#### Create
- the database db_name
- the user user_name
- and its password (user_pass): 
```
CREATE DATABASE db_name CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci; CREATE USER user_name@localhost identified by 'user_pass'; GRANT ALL PRIVILEGES on db_name.* to user_name@localhost; FLUSH privileges; quit;
```

Verify the transaction isolation level is set to READ_Commit and the collation is set to UTF8MB4 properly: 
```
mysql -h localhost -uroot -p -e "SELECT @@TX_ISOLATION; SELECT SCHEMA_NAME 'database', default_character_set_name 'charset', DEFAULT_COLLATION_NAME 'collation' FROM information_schema.SCHEMATA WHERE SCHEMA_NAME='db_name'"
```

The resultset should consist of “READ-COMMITTED” and “utf8mb4_general_ci” … go ahead with the installation of the redis-server.

### <a name="section5"></a> 5. Install and configure Redis

Install the redis-server to optimize application performance and to minimize the load on the database:
```
apt update
apt install redis-server php-redis -y
```

Change the redis configuration and set the group membership properly:
```
cp /etc/redis/redis.conf /etc/redis/redis.conf.bak
sed -i "s/port 6379/port 0/" /etc/redis/redis.conf
sed -i s/\#\ unixsocket/\unixsocket/g /etc/redis/redis.conf
sed -i "s/unixsocketperm 700/unixsocketperm 770/" /etc/redis/redis.conf
sed -i "s/# maxclients 10000/maxclients 512/" /etc/redis/redis.conf
usermod -aG redis www-data
```
```
cp /etc/sysctl.conf /etc/sysctl.conf.bak
sed -i '$avm.overcommit_memory = 1' /etc/sysctl.conf
```

Now reboot your server once:
```
reboot now
```

We are now ready to install our Symfony application.

### <a name="section6"></a> 6. Prepare NGINX for web application

First create all the configuration and webhost files (aka vhost). Change into sudo mode and create the /etc/nginx/conf.d/app.conf (vhost):

```
sudo -s
```
```
[ -f /etc/nginx/conf.d/default.conf ] && mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.bak
```
```
touch /etc/nginx/conf.d/default.conf
```
```
nano /etc/nginx/conf.d/app.conf
```

Paste all the following rows and customize the red marked parameters: 
```
server {
server_name your.domain.io;
listen 80 default_server;
listen [::]:80 default_server;
location ^~ /.well-known/acme-challenge {
proxy_pass http://127.0.0.1:81;
proxy_set_header Host $host;
}
location / {
return 301 https://$host$request_uri;
}
}
server {
server_name your.domain.io;
listen 443 ssl http2 default_server;
listen [::]:443 ssl http2 default_server;
root /var/www/app_name/;
location = /robots.txt {
allow all;
log_not_found off;
access_log off;
}
location = /.well-known/carddav {
return 301 $scheme://$host/remote.php/dav;
}
location = /.well-known/caldav {
return 301 $scheme://$host/remote.php/dav;
}
#SOCIAL app enabled? Please uncomment the following row
#rewrite ^/.well-known/webfinger /public.php?service=webfinger last;
#WEBFINGER app enabled? Please uncomment the following two rows.
#rewrite ^/.well-known/host-meta /public.php?service=host-meta last;
#rewrite ^/.well-known/host-meta.json /public.php?service=host-meta-json last;
client_max_body_size 10240M;
location / {
rewrite ^ /index.php;
}
location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)/ {
deny all;
}
location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console) {
deny all;
}
location ^~ /apps/rainloop/app/data {
deny all;
}
location ~ \.(?:flv|mp4|mov|m4a)$ {
mp4;
mp4_buffer_size 100M;
mp4_max_buffer_size 1024M;
fastcgi_split_path_info ^(.+?.php)(\/.*|)$;
set $path_info $fastcgi_path_info;
try_files $fastcgi_script_name =404;
include fastcgi_params;
include php_optimization.conf;
}
location ~ ^\/(?:index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|oc[ms]-provider\/.+).php(?:$|\/) {
fastcgi_split_path_info ^(.+?.php)(\/.*|)$;
set $path_info $fastcgi_path_info;
try_files $fastcgi_script_name =404;
include fastcgi_params;
include php_optimization.conf;
}
location ~ ^\/(?:updater|oc[ms]-provider)(?:$|\/) {
try_files $uri/ =404;
index index.php;
}
location ~ \.(?:css|js|woff2?|svg|gif|map|png|html|ttf|ico|jpg|jpeg)$ {
try_files $uri /index.php$request_uri;
access_log off;
expires 360d;
}
}
```
```
git clone git@github.com:guthub_username/repository_name.git
```
```
composer install
```
```

```
```

```
```

```
```

```
```

```