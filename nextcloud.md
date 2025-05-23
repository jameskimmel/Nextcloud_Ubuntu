# Example installation on Ubuntu 24.04.01 LTS with Apache2, PHP FPM, APCu, redis, and MariaDB, no Docker, no Snap

## Who is this for?
This is an example installation for Ubuntu users who want to host a Nextcloud instance bare metal. No Docker, no Snap.  
The goal of this guide is to have **no warnings in the admin center** and the instance should get a **perfect security score** from scan.nextcloud.com. The official documentation is pretty good, but it can be a little bit overwhelming to newcomers because you need to jump from one topic to another and have to read up on multiple things. This guide hopefully offers you a more streamlined experience.  
There are some placeholder values or variables that always start with x_. You need to replace them with your data.  
This is the structure of the setup used in this guide.

![Unbenanntes Diagramm drawio](https://github.com/jameskimmel/Nextcloud_Ubuntu/assets/17176225/fa1cdce1-7e24-4cdf-8845-b354a0a85b71)

## Network requirements

If you want to access Nextcloud remotely and share files with external users, there are some network requirements.  
You need a real, public routable, none Carrier-grade NAT (CG-NAT) IPv4 address.  
Don't know what CG-NAT is?  
[Test if you have a CG-NAT IPv4](https://github.com/jameskimmel/opinions_about_tech_stuff/blob/main/network%20stuff/CG-NAT.md).  
If you don't have a real IPv4 address, you could try to ask your ISP. Some ISPs will give you one for free, others charge you 5$ a month. Some call it "Gaming IP" or "NAS IP". You can also use IPv6 or a VPN instead. But if you want to share files externally with other users, only having IPv6 isn't great, since you don't know if all external users support IPv6.  
You also need split DNS described in the next paragraph.  

### Split DNS or Hairpin NAT
What is split DNS and why is it needed for IPv4?  
Let's assume your WAN IPv4 is 85.29.10.1 and your Nextcloud instance has the IP 192.168.1.10 and your domain is cloud.yourdomain.com.  
If you are on the road and try to connect to your Nextcloud, your client will ask "Hey what IP is cloud.yourdomain.com?" a DNS server will answer with "85.29.10.1".  
Then traffic will go to your firewall and some kind of **NAT** will redirect it to your Nextcloud instance on 192.168.1.10.  
But if you are on your local network, that will not work, because your firewall only NATs from WAN to LAN and not LAN to LAN.  
The easiest way to solve this is to use split DNS. Tell your DNS server, that instead of answering cloud.yourdomain.com with 85.29.10.1, it should answer it with 192.168.1.10. This is done by unbound overrides. Most home routers don't offer unbound. You may need to look into setting up a pi-hole DNS server that offers these overrides.  
Another option that should work (but I have not looked into it!) is Hairpin NAT.  

### IPv6
IPv6 works out of the box, because there is no pesky **NAT** involved. IPv6 does not need NAT, because every device gets its own public IP.  
You can enable DHCP6 during the Ubuntu installation, by setting it to DHCP6 or later on by adding dhcp6: true to netplan.  
Your host will not only one but get three IPv6.  
First one is a privacy extension enabled IPv6. Don't use that one, because it isn't static and will change. Second one is static, this is the one you want to use for nextcloud. Third one is only for local networks.  

### Optional: HTTP Strict Transport Security (HSTS)
This  is optional.
You can preload HTTP Strict Transport Security (HSTS) for your domain and all your subdomains.
That way you gain security by forcing all your domains and subdomains to use HTTPS. 
To learn more about HSTS and how you can enable it for your domain, go to https://hstspreload.org/

## Getting ready
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

## Install packages
Different versions of Ubuntu may have differing versions of PHP, for example Ubuntu 24.04 ships PHP 8.3.6, which is by pure luck the currently recommended version by Nextcloud. If the PHP version does not match, you can add other repositories. Adding  repositories to apt's sources (e.g. Sury's [ppa for Ubuntu](https://launchpad.net/~ondrej/+archive/ubuntu/php/) or [dpa for Debian](https://packages.sury.org/php/)) is beyond the scope of this tutorial.  
I think it is simpler to use the Ubuntu PHP version, but adding a PPA is also not that hard. The choice is yours :)
For up to date system requirements, please visit [Nextcloud admin manual](https://docs.nextcloud.com/server/latest/admin_manual/installation/system_requirements.html)  

We install all the software that is needed plus some optional software that is needed so we won't get warnings in the Nextcloud Admin Center.  
Ubuntu 24.04 comes with MariaDB 10.11.8 which is currently the recommended version. 
```bash
sudo apt install apache2 \
  bzip2 \
  exif \
  imagemagick \
  mariadb-server \
  redis-server
```

Install required php modules.
```bash
sudo apt install php-common \
  php-curl \
  php-xml \
  php-gd \
  php-imagick \
  php-mbstring \
  php-zip
```

install DB connector
```bash
sudo apt install php-mysql
```

install recommended modules
```bash
sudo apt install php-intl
```

install performance modules
```bash
sudo apt install php-apcu \
  php-redis 
```
install these for passwordless logins and performance:
```bash
sudo apt install php-bcmath \
  php-gmp 
```

optionally you could install ffmpeg (videos) and LibreOffice (Word, Excel, PowerPoint) for preview generation. Beware, these are pretty big.
```bash
sudo apt install ffmpeg \
  libreoffice 
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
exit and save (Ctrl + x and Y).

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
```
you should see a table with the text "READ-COMMITTED".  
```mysql
SELECT @@global.binlog_format;
```
now you should see the "ROW".  

If everything looks good, we can continue. 
Insert this to create a database called nextcloud. Replace all three x_ variables with your data.
```mysql
CREATE USER 'x_database_user'@'localhost' IDENTIFIED BY 'x_database_password';
CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
GRANT ALL PRIVILEGES ON nextcloud.* TO 'x_database_user'@'localhost';
FLUSH PRIVILEGES;
exit;
```
You should see 4 times a "Query OK" line and a "Bye" at the end.

## Nextcloud
Download Nextcloud
```bash
wget https://download.nextcloud.com/server/releases/latest.tar.bz2
wget https://download.nextcloud.com/server/releases/latest.tar.bz2.sha256
```

verify
```bash
sha256sum -c --ignore-missing latest.tar.bz2.sha256 
```
should show OK.  

Extract and move to the webroot. Change ownership and delete install files
```bash
tar -xjvf latest.tar.bz2
sudo mv nextcloud /var/www
sudo chown -R www-data:www-data /var/www/nextcloud
rm  latest.tar.bz2 latest.tar.bz2.sha256
```

## PHP FPM

We stop apache, install FPM and enable the modules. Replace 8.3 with newer version if needed.
```bash
sudo systemctl stop apache2
sudo apt install php-fpm
sudo a2enmod proxy_fcgi setenvif
sudo a2enconf php8.3-fpm
```

We set the hostname of our server. This should be a FQDN.
```bash
sudo hostnamectl set-hostname cloud.x_youdomain.com
```

Test the config
```bash
sudo apachectl configtest
```
You should see "Syntax OK".  

restart apache
```bash
sudo systemctl restart apache2
```

In the next steps, we enable MPM event.

```bash
sudo systemctl stop apache2
```
```bash
sudo a2dismod php8.3
```
should not exist

```bash
sudo a2dismod mpm_prefork
```
should already be disabled

```bash
sudo a2enmod mpm_event
```
should be enabled already

```bash
sudo a2enconf php8.3-fpm
```
should be enabled already

```bash
sudo a2enmod proxy
```
should be enabled already

```bash
sudo a2enmod proxy_fcgi
```
should be enabled already

Test the config
```bash
sudo apachectl configtest
```
You should "Syntak OK".

restart apache
```bash
sudo systemctl restart apache2
```

check your config. 
```bash
sudo apachectl -M | grep 'mpm'
```
should output 
```bash
Output
mpm_event_module (shared)
```
and 
```bash
sudo apachectl -M | grep 'proxy'
```
should output 
```bash
Output
proxy_module (shared)
proxy_fcgi_module (shared)
```

It is now time to check if PHP is using the FastCGI Process Manager. To do so you’ll create a small PHP file that will show you all the information related to PHP.

```bash
sudo nano /var/www/html/info.php
```
insert this and save and exit
```bash
<?php phpinfo(); ?>
```

In your browser, enter the IP of your Nextcloud host (for example http://192.168.1.10) and you should see the Apache2 default page. 
Add /info.php to the end (http://x_nextcloud_host_IPv4/info.php) and you can see the PHP infos. The fourth line should be "Server API" with "FPM/FastCGI". 

## PHP settings
We wanna change the PHP memory limit and upload filesize. Replace 8.3 if you have a newer version of PHP.
```bash
sudo nano /etc/php/8.3/fpm/php.ini
```

We search for these settings to change (use ctrl+W to search in nano). Watch out to delete the ; before the opcache settings, otherwise they are commented out. 
```bash
memory_limit = 1G
upload_max_filesize = 50G
post_max_size = 0
max_execution_time = 300
date.timezone = Europe/Amsterdam
opcache.memory_consumption=256
opcache.interned_strings_buffer=32
```

Save and exit. Reload FPM
```bash
sudo systemctl reload php8.3-fpm.service 
```
after reloading the webpage, you should see the changes in info.php.

PHP-FPM default values are to low. Find appropiate values with this tool https://spot13.com/pmcalculator/. 
For me this is this:
pm.max_children = 201
pm.start_servers = 50
pm.min_spare_servers = 50
pm.max_spare_servers = 50

and insert them here
```bash
sudo nano /etc/php/8.3/fpm/pool.d/www.conf
```

in the same file, uncomment these environment variables:
```bash
;env[HOSTNAME] = $HOSTNAME
;env[PATH] = /usr/local/bin:/usr/bin:/bin
;env[TMP] = /tmp
;env[TMPDIR] = /tmp
;env[TEMP] = /tmp
```
You should see these changes on the webpage after you restart FPM
```bash
sudo systemctl reload php8.3-fpm.service 
```

## Apache2
Create the nextcloud folder
```bash
sudo -u www-data mkdir /var/www/nextcloud/data
```

Configure Apache2
```bash
sudo nano /etc/apache2/sites-available/nextcloud.conf
```
insert and change the x variable:
```bash
<VirtualHost *:80>
  DocumentRoot /var/www/nextcloud/
  ServerName  cloud.x_youromain.com

  <Directory /var/www/nextcloud/>
    Satisfy Any
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
sudo a2enmod rewrite headers env dir mime setenvif
sudo systemctl restart apache2
```

We disable the default page and delete the default folder.
```bash
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
sudo rm -r /var/www/html
```

Test the config
```bash
sudo apachectl configtest
```

## Certbot
This guide assumes you have Certbot installed. If you dont have it installed yet, 
Certbot currently recommends installing it by using snap.
```bash
sudo apt install snapd
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

For certbot to be sucessfull, you need an A or AAAA record that points to your instance with the open port 80.

```bash
sudo certbot --apache
```
Follow the certbot instructions. When asked if http should be redirected or not, say yes.
If you have done everything right, it should automatically detect your hostname from the Apache2 config. 

If you are unable to get a cert, most of the time there is something wrong with your firewall or DNS settings. From the outside, cloud.yourdomain.com should point to the IP of your nextcloud host. For IPv6 you need a firewall rule to allow traffic on port 80 to your nextcloud IPv6.  
For IPv4, you need to NAT incoming port 80 traffic to your nextcloud IPv4.
You can use webpages to check if your ports are open. Like [DNSchecker.org](https://dnschecker.org/port-scanner.php).   

To test if the automatic removal is working run
```bash
sudo certbot renew --dry-run
```

## Install Nextcloud 
You can install Nextcloud via the webGUI over IP, over the domain name or in the console. 
I would recommend you using your domain name. That way you can save some work and test if your DNS is working.
On the webGUI you can define a admin username and password. 
We insert x_database_user and x_database_password and also the database name "nextcloud".
Host we can leave at "localhost". Then click on install.

If you wan't to use CLI, use these commands:
```bash
cd /var/www/nextcloud/
sudo -u www-data php occ  maintenance:install \
--database='mysql' --database-name='nextcloud' \
--database-user='x_database_user' --database-pass='x_database_password' \
--admin-user='x_nextcloud_admin_user' --admin-pass='x_nextcloud_admin_user_password' \
--data-dir='/var/www/nextcloud/data'
```

Congrats, we now have a working Nextcloud instance! 
Depending on which you maybe now see a trusted domains error. That is fine.
Now it needs security tweaks and some fine tuning!

## PHP config settings
Depending if you used the CLI or webpage, some values maybe already correct.

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

we also change the overrides:
```bash
  'overwrite.cli.url' => 'https://cloud.x_youromain.com',
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
sudo -u www-data php /var/www/nextcloud/occ maintenance:update:htaccess
```
Now you should be able to access cloud.x_youromain.com without untrusted domain warning. 
If the warning is still there, try to access it over your mobile to rule out an error in your local DNS settings. 

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
also uncomment and set 
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
Change nextcloud PHP config. 
While we are in this file, we also add the memcache.local for APCu.

```bash
sudo nano /var/www/nextcloud/config/config.php
```
Add:
```PHP
  'memcache.local' => '\OC\Memcache\APCu',
  'memcache.locking' => '\OC\Memcache\Redis',
  'redis' =>
  array (
   'host'     => '/run/redis/redis-server.sock',
   'port'     => 0,
  ),
```

### APCu
Change apcu.ini. Watch out for the PHP version
```bash
sudo nano /etc/php/8.3/fpm/conf.d/20-apcu.ini 
```
Change it to: 
```config
extension=apcu.so
apc.enabled=1
apc.enable_cli=1
```
To start APCu automatically use this command and replace the PHP version 8.3 if needed
```bash
sudo -u www-data php --define apc.enable_cli=1  /var/www/nextcloud/occ  maintenance:repair
```

Check if Opcache is working
```bash
php -r 'phpinfo();' | grep opcache.enable
```

## Configure Apache2 HSTS
We set the strict transport security. 

```bash
sudo nano /etc/apache2/sites-available/nextcloud-le-ssl.conf
```
Insert the IfModule mod_headers.
Your setting should look like this:

```bash
<IfModule mod_ssl.c>
<VirtualHost *:443>
  DocumentRoot /var/www/nextcloud/
  ServerName  cloud.x_yourdomain.com

  <Directory /var/www/nextcloud/>
    Satisfy Any
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
Pretty URLs remove the index.php-part in all Nextcloud URLs, for example in sharing links like https://cloud.yourdomain.com/index.php/s/Sv1b7krAUqmF8QQ, making URLs shorter and thus prettier.

```bash
sudo nano /var/www/nextcloud/config/config.php
```
```php
  'htaccess.RewriteBase' => '/',
```
save and exit.
update htaccess
```bash
sudo -u www-data php /var/www/nextcloud/occ maintenance:update:htaccess
```

### maintenance window
We can define when a the maintenance window starts (UTC time!). By default, the maintenance windows ends 4 hours after the start. 
We start it at 1 in the morning.
```bash
sudo -u www-data php /var/www/nextcloud/occ config:system:set maintenance_window_start --type=integer --value=1
```

### mimetype and indizes
```bash
sudo -u www-data php /var/www/nextcloud/occ maintenance:repair --include-expensive
sudo -u www-data php /var/www/nextcloud/occ db:add-missing-indices
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
Updates can take a long time if your data directory is on a NFS share. 
To speed it up, create an update directory that is on your local SSD instead. 
```bash
sudo mkdir /var/www/updateDirNextcloud
sudo chown www-data:www-data /var/www/updateDirNextcloud
sudo nano /var/www/nextcloud/config/config.php
```
add the line
```bash
  'updatedirectory' => '/var/www/updateDirNextcloud',
```
