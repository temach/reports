# LS Lab 3 Monitoring

Artem Abramov

### Task 1 - Preparation

I choose Zabbix . 

##### Install a new guest system

Installed guest on Xen. I wanted to get over monitoring first to I quickly setup a test server with python built-in httpd.

```
python3 -m http.server 8000 &> /dev/null
```



##### Install and setup web-server + DBMS (e.g. NGINX + MariaDB)

##### Install and setup a CMS. Create a page with some heavy content

##### Install another guest system and deploy the monitoring facility (and agents, if required)

I setup another Xen guest. And configured MariaDB + PHP + NGinx + Zabbix

sources: 

- https://www.dmosk.ru/miniinstruktions.php?mini=zabbix-server-ubuntu#zabbix
- https://serveradmin.ru/ustanovka-i-nastroyka-zabbix-4-0/#_Zabbix_40_Ubuntu_Debian

DB install and set root password:

```
apt-get install mariadb-server
systemctl enable mariadb
systemctl start mariadb
mysqladmin -u root password
```

Web server:

```
apt-get install nginx
systemctl enable nginx
systemctl start nginx
```

Check that nginx works by going to the ip address in browser.

PHP:

```
apt-get install php php-fpm php-mysql php-pear php-cgi php-common php-ldap php-mbstring php-snmp php-gd php-xml php-gettext php-bcmath
```

Configure php (get version with `php -v`):

```
# cat /etc/php/7.2/fpm/php.ini
date.timezone = "Europe/Moscow"
max_execution_time = 300
post_max_size = 16M
max_input_time = 300
max_input_vars = 10000
```

Enable:

```
systemctl enable php7.2-fpm
systemctl restart php7.2-fpm
```

Configure php in nginx:

```
# cat /etc/nginx/sites-enabled/default
    ...
    location / {
        index  index.php;
        ...
    }
    ...
    location ~ \.php$ {
        set $root_path /var/www/html;
        fastcgi_buffer_size 32k;
        fastcgi_buffers 4 32k;
        fastcgi_pass unix:/run/php/php7.2-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $root_path$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_param DOCUMENT_ROOT $root_path;
    }
    ...
```

Check nginx config and start:

```
nginx -t
systemctl restart nginx
```

Check PHP works, create index.php:

```
# cat /var/www/html/index.php
<?php phpinfo(); ?>
```

Check by going to the IP of the server (should display PHP version info).

Zabbix add repo (note use 4.0, not 4.2, not 4.4):

```
wget https://repo.zabbix.com/zabbix/4.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_4.0-3+bionic_all.deb
```

Install and add packages:

```
dpkg -i zabbix-release_4.0-3+bionic_all.deb
apt update && apt upgrade
apt install zabbix-server-mysql zabbix-frontend-php
```

At this point make sure to disable apache2:

```
systemctl stop apache2
systemctl disable apache2
```

Configure mysql for zabbix:

source:  https://www.zabbix.com/documentation/4.0/manual/appendix/install/db_scripts#mysql

Login:

```
mysql -uroot -p
> create database zabbix character set utf8 collate utf8_bin;
> grant all privileges on zabbix.* to zabbix@localhost identified by 'zabpassword';
> \q
```

Its important to create db as utf-8 else faced with error about string too long.

Now apply zabbix schema to db, better take it step by step:

```
cd /usr/share/doc/zabbix-server-mysql/
gunzip create.sql.gz
mysql -v -u root -p zabbix < create.sql
```

Enter the password from sql above `identified by 'zabpassword';` i.e. `zabpassword`.

Edit zabbix config, first critical options:

```
# cat /etc/zabbix/zabbix_server.conf
DBName=zabbix
DBUser=zabbix
DBPassword=zabpassword
...
```

Then semi-critical options:

```
DBHost=localhost
ListenIP=0.0.0.0
Timeout=10
```

Make sure log and config dirs are created:

```
mkdir -p /etc/zabbix/zabbix_server.conf.d
mkdir -p /var/log/zabbix-server
chown zabbix:zabbix /var/log/zabbix-server
```

Start and enable:

```
systemctl enable zabbix-server
systemctl start zabbix-server
```

Next is merging zabbix with nginx because zabbix web frontend gets installed into ` /usr/share/zabbix`

```
# cat /etc/nginx/sites-enabled/default
...
root /usr/share/zabbix;
...
set $root_path /usr/share/zabbix;
...
```

Check nginx config and restart:

```
nginx -t
systemctl restart nginx
```

Check the ip address for web interface:

![Installation - Mozilla Firefox_001](LS-Lab-3-monitoring.assets/Installation%20-%20Mozilla%20Firefox_001.png)