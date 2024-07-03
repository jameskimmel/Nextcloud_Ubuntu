# Example installation on Ubuntu 22.04.03 LTS with Apache2, APCu, redis, and MariaDB behind an NGINX proxy, no Docker, no Snap

## Who is this for?
This is an example installation for Ubuntu users who want to host a Nextcloud instance bare metal. No Docker, no Snap.
The goal of this guide is to have **no warnings in the admin center** and the instance should get a **perfect security score** from scan.nextcloud.com. The official documentation is pretty good, but it can be a little bit overwhelming to newcomers because you need to jump from one topic to another and have to read up on multiple things. This guide should offer you a more streamlined experience.

There are some placeholder values or variables that always start with x_. You need to replace them with your data. 
This is the structure of the setup used in this guide.

![Unbenanntes Diagramm drawio](https://github.com/jameskimmel/Nextcloud_Ubuntu/assets/17176225/fa1cdce1-7e24-4cdf-8845-b354a0a85b71)

## Network requirements

If you want to host Nextcloud in your home and want to access it remotely or even share some files externally, there are some network requirements. 
You need a real, public routable, none Carrier-grade NAT (CG-NAT) IPv4 address. 
If you don't have a real IPv4 address, you could ask your ISP to give you one. Some ISPs will give you one for free, others charge you 5$ a month. Some call it "Gaming IP" or "NAS IP". You could also use IPv6 or a VPN instead. If you want to share files externally, only having IPv6 isn't great, since you don't know if all external users are able to use IPv6.  
You also need split DNS described in the next paragraph.

### Split DNS or Hairpin NAT
Why is split DNS this necessary? 
Let's assume your WAN IPv4 is 85.29.10.1 and your Nextcloud instance has the IP 192.168.1.10 and your domain is cloud.yourdomain.com. 
If you are on the road and try to connect to your Nextcloud, your client will ask "Hey what IP is cloud.yourdomain.com?" a DNS server will answer with "85.29.10.1".
Then traffic will go to your firewall and some kind of NAT will redirect it to your Nextcloud instance on 192.168.1.10. 
But if you are on your local network, that probably will not work, because your firewall only NATs from WAN to LAN and not LAN to LAN. 
The easiest way to solve this is to use split DNS. Tell your DNS server, that instead of answering cloud.yourdomain.com with 85.29.10.1 it should answer it with 192.168.1.10. This is done by unbound overrides. Most home routers don't offer unbound, so you may need to look into setting up a pi-hole DNS server.  
Another option that should work (but I have not looked into it!) is Hairpin NAT.

## Getting ready
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

## Install packages
Different versions of Ubuntu may have differing versions of PHP, for example Ubuntu 22.04 ships PHP 8.1, which is not the currently recommended version by Nextcloud. Nextcloud currently recommends PHP 8.2. Adding optional repositories to apt's sources (e.g. Sury's [ppa for Ubuntu](https://launchpad.net/~ondrej/+archive/ubuntu/php/) or [dpa for Debian](https://packages.sury.org/php/)) is beyond the scope of this tutorial, and will require modifying the name of libapache2-mod-php and all php modules to include the specific version number, e.g. libapache2-mod-php8.2 php8.2-apcu php8.2-bcmat and so on.I think it is simpler to use the Ubuntu PHP version, but adding a PPA is also not that hard. The choice is yours :)
In this tutorial we will use the included PHP packages from Ubuntu. While 8.1 is not recommended, it is still currently supported by Nextcloud. 
For up to date system requirements, please visit [Nextcloud admin manual](https://docs.nextcloud.com/server/latest/admin_manual/installation/system_requirements.html)  
We install all the software that is needed plus some optional software so we won't get warnings in the Nextcloud Admin Center.

```bash
sudo apt install apache2 \
  bzip2 \
  exif \
  imagemagick \
  mariadb-server \
  redis-server
```

Install all php modules.
```bash
sudo apt install libapache2-mod-php \
  php-apcu \
  php-bcmath \
  php-bz2 \
  php-ctype \
  php-curl \
  php-dom \
  php-gd \
  php-gmp \
  php-imagick \
  php-intl \
  php-mbstring \
  php-mysql \
  php-posix \
  php-redis \
  php-xml \
  php-zip
```
## MariaDB

Change the MariaDB settings to the recommended READ-COMITTED and binlog format ROW.
```bash
sudo nano /etc/mysql/conf.d/nextcloud.cnf
```
insert
```bash
[mysqld]
transaction_isolation = READ-COMMITTED
binlog_format = ROW
```
exit and save.
Reload mariadb

```bash
sudo systemctl restart mariadb.service
```

Secure MariaDB. Insert a root password, otherwise just use the defaults by pressing enter.
```bash
sudo mariadb-secure-installation
```

Create the database
```bash
sudo mariadb
```
You should now see "MariaDB [(none)]>"

Check if the tx_isolation is "READ-COMITTED" and if binlog_format is "ROW".
```mysql
SELECT @@global.tx_isolation;
SELECT @@global.binlog_format;
```
If everything looks good, we can continue. 
Insert this to create a database called nextcloud. Replace all three x_ variables with your data.
```mysql
CREATE USER 'x_database_user'@'localhost' IDENTIFIED BY 'x_database_password';
CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
GRANT ALL PRIVILEGES ON nextcloud.* TO 'x_database_user'@'localhost';
FLUSH PRIVILEGES;
exit;
```
You should see 3 times a "Query OK" line and a "Bye" at the end.

## Nextcloud
Download Nextcloud
```bash
wget https://download.nextcloud.com/server/releases/latest.tar.bz2
wget https://download.nextcloud.com/server/releases/latest.tar.bz2.asc
wget https://download.nextcloud.com/server/releases/latest.tar.bz2.md5
wget https://nextcloud.com/nextcloud.asc
gpg --import nextcloud.asc
```
verify
```bash
md5sum -c latest.tar.bz2.md5 < latest.tar.bz2
```
and
```bash
gpg --verify latest.tar.bz2.asc latest.tar.bz2
```
extract and move to the webroot. Change ownership and delete install files
```bash
tar -xjvf latest.tar.bz2
sudo cp -r nextcloud /var/www
sudo chown -R www-data:www-data /var/www/nextcloud
rm -r nextcloud
rm  latest.tar.bz2 latest.tar.bz2.asc  latest.tar.bz2.md5  nextcloud.asc
```
## PHP settings

We wanna change the PHP memory limit and upload filesize. Replace 8.1 if you have a newer version of PHP.
```bash
sudo nano /etc/php/8.1/apache2/php.ini
```

We search for these settings to change (use ctrl+W to search in nano).
```bash
memory_limit = 1G
upload_max_filesize = 50G
post_max_size = 0
max_execution_time = 3600
date.timezone = Europe/Amsterdam
opcache.interned_strings_buffer=16
```
Save and exit. Reload apache2
  
```bash
sudo systemctl reload apache2.service
```
create a PHP file so we can check the changes
```bash
sudo nano /var/www/html/phpinfo.php
```
and insert

```PHP
<?php phpinfo(); ?>
```

Now can see the default Apache2 page at 
http://x_nextcloud_host_IPv4.
To see the PHP settings go to 
http://x_nextcloud_host_IPv4/phpinfo.php
You should find the values we defined. 
To disable that page and delete the html folder, run 
```bash
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
sudo rm -r /var/www/html
```
## Apache2
Create the data folder. You can also use a different location.
Just make sure to replace /var/www/nextcloud/data everywhere with your data path. 
```bash
sudo mkdir /var/www/nextcloud/data
sudo chown -R www-data:www-data /var/www/nextcloud/data
```

Configure Apache2
```bash
sudo nano /etc/apache2/sites-available/nextcloud.conf
```
insert:
```bash
<VirtualHost *:80>
  DocumentRoot /var/www/nextcloud/
  ServerName  cloud.x_youromain.com

  <Directory /var/www/nextcloud/>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews

    <IfModule mod_dav.c>
      Dav off
    </IfModule>
  </Directory>
</VirtualHost>
```

Enable site and mods:
```bash
sudo a2ensite nextcloud.conf
sudo a2enmod rewrite
sudo a2enmod headers
sudo a2enmod env
sudo a2enmod dir
sudo a2enmod mime
sudo service apache2 restart
```

## Cerbot
This guide assumes you have Certbot installed. If you dont have it installed yet, here is the currently recommended way to do it:
```bash
sudo apt install snapd
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

For certbot to be sucessfull, you need an A or AAAA record that points to your instance with the open port 80.

```bash
sudo certbot
```
Follow the certbot instructions.
If you have done everything right, it should automatically detect hostnames from the Apache2 configs. 
If you are unable to get a cert, most of the time there is something wrong with your firewall opening the port 80 or your DNS settings.
Certbot will create a cert and also change your config to redirect all traffic to https.
To test if the automatic removal is working run
```bash
sudo certbot renew --dry-run
```

## Enable SSL
```bash
sudo a2enmod ssl
sudo a2ensite default-ssl
sudo service apache2 reload
```

## Install Nextcloud 
You can install Nextcloud with that command or by using the WebPage.

```bash
cd /var/www/nextcloud/
sudo -u www-data php occ  maintenance:install \
--database='mysql' --database-name='nextcloud' \
--database-user='x_database_user' --database-pass='x_database_password' \
--admin-user='x_nextcloud_admin_user' --admin-pass='x_nextcloud_admin_user_password' \
--data-dir='/var/www/nextcloud/data'
```

If we navigate now to https://cloud.x_youromain.com, we should see a warning that try to connect over an untrusted domain. 

## PHP config settings
Edit config.php file. 
```bash
sudo nano /var/www/nextcloud/config/config.php
```
Set the trusted_domains array
```bash
  'trusted_domains' =>
  array (
    0 => 'cloud.x_youromain.com',
  ),
```
while we are at it, you could also add these settings to match your locales:
```bash
'default_language' => 'de',
'default_locale' => 'de_DE',
'default_phone_region' => 'DE',
```
save and exit

update the settings by
```bash
cd /var/www/nextcloud/
sudo -u www-data php occ maintenance:update:htaccess
```
Now you should be able to access cloud.x_youromain.com without untrusted domain warning. If the warning is still there, try to access it over your mobile networt to rule out an error in your local DNS settings. 

## Set up crontab
We wanna use crontab instead of AJAX.
```bash
sudo crontab -u www-data -e
```
press 1 to use nano and insert at the end
```bash
*/5  *  *  *  * php -f /var/www/nextcloud/cron.php
```
Change the settings by
```bash
sudo -u www-data php /var/www/nextcloud/occ background:cron
```
## Mail
In the web GUI, go to user settings and insert the mail address for the x_nextcloud_admin_user you created earlier. 
Naviate to Administration -> Basic settings to set up outgoing mail. In my example, I am using Office365 with an AppPassword created in the account settings of MS365.
```config
Servername: smtp.office365.com
Port: 587
Encryption: STARTTLS
Needs authentification, sender and user is me@mydomain.com 
AppPasswort
```

## Caching

### Redis
Add redis to the www-data group
```bash
sudo usermod -a -G redis www-data
```
Configure Redis server
```bash
sudo nano /etc/redis/redis.conf
```
uncomment 
unixsocket /run/redis/redis-server.sock
Set 
unixsocketperm to 770
Exit and save.
Restart redis
```bash
sudo service redis-server restart
```
Check output of redis
```bash
ls -lh /var/run/redis
```
Change nextcloud PHP config. While we are in this file, we also add the memcache.local for APCu.

```bash
sudo nano /var/www/nextcloud/config/config.php
```
Add:
```PHP
  'memcache.local' => '\OC\Memcache\APCu',
  'memcache.locking' => '\OC\Memcache\Redis',
  'redis' => array(
     'host' => 'localhost',
     'port' => 6379,
     'timeout' => 1,
     'password' => '',  
      ),
```

### APCu
Change apcu.ini. Watch out for the PHP version
```bash
sudo nano /etc/php/8.1/apache2/conf.d/20-apcu.ini
```
Change it to: 
```config
extension=apcu.so
apc.enabled=1
apc.enable_cli=1
```
To start APCu automatically use this command and replace the PHP version 8.1 if needed
```bash
sudo -u www-data php8.1 --define apc.enable_cli=1  /var/www/nextcloud/occ  maintenance:repair
```

Check if Opcache is working
```bash
php -r 'phpinfo();' | grep opcache.enable
```

## HTTP Strict Transport Security (HSTS)
This step is optional.
You can preload HTTP Strict Transport Security (HSTS) for your domain and all your subdomains.
That way you gain security by forcing all your domains and subdomains to use HTTPS. 
To learn more about HSTS and how you can enable it for your domain, go to https://hstspreload.org/
If you don't want to use this, you need to make a small change in Apache and NGINX by removing "preload". 

## Configure Apache2 HSTS
We set the strict transport security. 

```bash
sudo nano /etc/apache2/sites-available/nextcloud-le-ssl.conf
```
Insert the IfModule.
Your setting should look like this:

```bash
<IfModule mod_ssl.c>
<VirtualHost *:443>
  DocumentRoot /var/www/nextcloud/
  ServerName  cloud.yourdomain.com

  <Directory /var/www/nextcloud/>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews

    <IfModule mod_dav.c>
      Dav off
    </IfModule>

    <IfModule mod_headers.c>
      Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    </IfModule>

  </Directory>

SSLCertificateFile /etc/letsencrypt/live/cloud.yourdomain.com/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/cloud.yourdomain.com/privkey.pem
Include /etc/letsencrypt/options-ssl-apache.conf
</VirtualHost>
</IfModule>
```
save and exit. Reload
```bash
sudo systemctl reload apache2
```
If you decided against HSTS, ditch the "preload" in the IfModule on use it like this instead
```bash
      Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains"
```

### Pretty URLs 
Pretty URLs remove the index.php-part in all Nextcloud URLs, for example in sharing links like https://example.org/nextcloud/index.php/s/Sv1b7krAUqmF8QQ, making URLs shorter and thus prettier.

```bash
sudo nano /var/www/nextcloud/config/config.php
'overwrite.cli.url' => 'https://example.org/',
'htaccess.RewriteBase' => '/',
sudo -u www-data php /var/www/nextcloud/occ maintenance:update:htaccess
```
### maintenance window
We can define when a the maintenance window starts (UTC time!). By default, the maintenance windows ends 4 hours after the start. 
We start it at 3 in the morning.
```bash
sudo -u www-data php /var/www/nextcloud/occ config:system:set maintenance_window_start --type=integer --value=1
```

Congrats! You should no have no warnings in the admin center and a perfect score on scan.nextcloud.com!

### NFS share as data directory (optional)
Instead of using local storage, you can move the data directory to a NFS mount. 
```bash
sudo apt install nfs-common
```
Create a folder and a mountpoint in fstab. 
In this example the data dir is set to /mnt/nextcloud/data. Make sure the user www-data has write access. 
```bash
sudo nano /var/www/nextcloud/config/config.php
```
add the line
```bash
'datadirectory' => '/mnt/nextcloud/data',
```
### temp path for Update (optional)
Updates can take a long time if your data directory is on a NFS share, because Nextcloud will create a backup first. 
To speed it up, create a temp backup and update dir.
```bash
sudo mkdir /var/www/update
sudo chown www-data:www-data /var/www/update/
sudo nano /var/www/nextcloud/config/config.php
```
add the line
```bash
'updatedirectory' => '/var/www/update',
```
