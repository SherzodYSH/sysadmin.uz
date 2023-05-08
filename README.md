# sysadmin.uz
Install Apache & MySQL

Most likely, you already have Apache2 and MySQL installed on your server. If not, you can quickly install them with

$ sudo apt update
$ sudo apt install apache2
$ sudo apt install mariadb-server

Before setting up a new user and database, you should secure your MySQL server (again, you can skip this step if you already have used MySQL before)

sudo mysql_secure_installation
VALIDATE PASSWORD COMPONENT: n
New root passwort: <YOUR MYSQL PASSWORD>
Re-enter new password: <YOUR MYSQL PASSWORD>
Remove anonymous user: y
Disallow root login remotely: y
Remove test database and access to it: y
Reload privileges tables now: y

------------------------------------------------

Prepare MySQL database

In a next step, you want to create a new user and database on your MySQL server:

sudo mysql -u root -p
<ENTER YOUR MYSQL PASSWORD>
mysql> create database nextcloud;
mysql> create user 'nextcloud'@'localhost' identified by 'password';
mysql> grant all privileges on nextcloud.* to 'nextcloud'@'localhost';
mysql> flush privileges;
mysql> quit


-----------------------------------------------


Install PHP for Nextcloud

For Nextcloud to work, you will need a new version of PHP as well as a few PHP libraries. Likely, you have already installed a version of PHP but its also no problem to have multiple versions installed on your server. We will talk about how to enable the latest installed PHP version in Apache in the section Configure Apache for Nextcloud below.

sudo apt install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt update && sudo apt upgrade

Install PHP with all required libraries:

sudo apt install php8.1 -y
sudo apt install php8.1-gd php8.1-mysql php8.1-curl php8.1-mbstring php8.1-apcu -y
sudo apt install php8.1-intl php8.1-gmp php8.1-bcmath php8.1-xml -y
sudo apt install libapache2-mod-php8.1 php8.1-zip php-imagick redis-server php-redis -y

------------------------------------------------

Download Nextcloud Hub 21

    Go to https://nextcloud.com
    Click on “Get Nextcloud”
    Click on “Server packages”
    Right-click on “Download Nextcloud”
    Click on “Copy link address”

Then, download the latest version of Nextcloud directly onto your home server:

wget https://download.nextcloud.com/server/releases/latest.zip

Next, unzip Nextcloud directly into the www directory of Apache (first run sudo apt install unzip if you don’t have unzip installed yet):

sudo unzip nextcloud-23.0.0.zip -d /var/www

Switch directories and change ownership of the Nextcloud folder to Apache, which is the “www-data” user:

cd /var/www
sudo chown -R www-data:www-data nextcloud/

-------------------------------------------------


Configure Apache for Nextcloud

*First, you need to enable a few Apache modifications for Nextcloud to properly run

sudo a2enmod headers env dir mime rewrite
sudo service apache2 restart

-----------------------------------------------


Add a VirtualHost entry in the default configuration file of Apache. If you already have a domain name, you can additionally specify it under the DocumentRoot directive. Don’t worry if you don’t have your own domain – you can just as easily use Nextcloud only locally without the need of a domain name.

/etc/apache2/sites-available/000-default.conf

<VirtualHost *:80>

    ServerName cloud.yourdomain.com
    DocumentRoot /var/www/nextcloud

    <Directory /var/www/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews

        <IfModule mod_dav.c>
            Dav off
        </IfModule>

        RewriteEngine On
        RewriteRule ^/\.well-known/carddav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
        RewriteRule ^/\.well-known/caldav https://%{SERVER_NAME}/remote.php/dav/ [R=301,L]
        RewriteRule ^/\.well-known/host-meta https://%{SERVER_NAME}/public.php?service=host-meta [QSA,L]
        RewriteRule ^/\.well-known/host-meta\.json https://%{SERVER_NAME}/public.php?service=host-meta-json [QSA,L]
        RewriteRule ^/\.well-known/webfinger https://%{SERVER_NAME}/public.php?service=webfinger [QSA,L]

    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>

Finally, restart Apache

sudo service apache2 restart

------------------------------------------------


In my case, my Nextcloud server is located at 192.168.1.50, hence I need to update the Nextcloud configuration file in order to be able to use that local IP address to browse to my Nextcloud instance. Also, this is where you can specify your domain if you have one:
Kamchiliklarni sozlash; Trsuted domain
/var/www/nextcloud/config/config.php
'trusted_domains' => 
array(
  0 => 192.168.1.50:80
  1 => http://yourdomain.com
),
'default_phone_region' => 'UZ',
'memcache.local' => '\\OC\\Memcache\\APCu',
),


--------------------------------------------------



PHP sozlamasi
/etc/php/8.1/apache2# nano php.ini
memory_limit = 512M
 
-------------------------------------------------

Fix php-imagick warning

On a fresh installation of Ubuntu 20.04 you will likely get the following warning message:

    Module php-imagick in this instance has no SVG support. For better compatibility it is recommended to install it.

Then you will have to first uninstall ImageMagick and all its dependencies using:

sudo apt remove imagemagick-6-common php-imagick
sudo apt autoremove

and then reinstall imagemagick;

sudo apt install imagemagick php-imagick

----------------------------------------------------

https://techguides.yt/guides/how-to-install-and-configure-nextcloud-hub-21/

-----------------------------------------------------

SSL

https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-ubuntu-16-04

nano /etc/apache2/sites-available/nextcloud1.conf

<VirtualHost *:80>
   ServerName 192.168.241.255
   Redirect / https://192.168.241.255/
</VirtualHost>

<VirtualHost *:443>
   ServerName 192.168.241.255
   DocumentRoot /var/www/nextcloud

   SSLEngine on
   SSLCertificateFile /etc/ssl/certs/apache-selfsigned.crt
   SSLCertificateKeyFile /etc/ssl/private/apache-selfsigned.key
</VirtualHost>

Strict-Transport-Security
<VirtualHost *:443>
  ServerName cloud.nextcloud.com
    <IfModule mod_headers.c>
      Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
    </IfModule>
 </VirtualHost>

