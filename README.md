# Magento-installation
## Overview:

- We will set up a single server for Magento 2 Open Source, with these services:
  -  Nginx
  -  php-fpm
  -  MySQL
  -  Varnish
  -  Redis
  -  OpenSearch

- Architecture as shown in the image bellow:
!<img width="1320" alt="Archeticture" src="https://github.com/user-attachments/assets/9f6de9ad-2ed5-44e3-b458-d8e00aa81ee9">

## Quick Start Guide:
- I have created a new user and a new path to work in.
- Magento will be ran by a new system user called magento. Let’s create a new system user now, execute this command below:
```
/usr/sbin/adduser \
   --system \
   --shell /bin/bash \
   --gecos 'Magento user' \
   --group \
   --home /opt/magento \
magento
```
- Then, let’s give the new user a password. You will be prompted to type the password for user ‘magento’ twice, the password will not be shown there in your screen: at my case the password is: 123456
```
passwd magento
```
Once done, we can give the new user a sudo privilege:
```
usermod -aG sudo magento
```
- Let’s switch to the new user now. From now on, the commands will be run by the new user:
```
su - magento
```

### Intsall php:
- Let’s install PHP 8.3 and its extensions:
```
sudo apt install php-{bcmath,common,curl,fpm,gd,intl,mbstring,mysql,soap,xml,xsl,zip,cli}
```
- Next, we need to modify the following settings in the php.ini file:
  
 Increase memory_limit to 512M

 Set short_open_tag to On
 
 Set upload_max_filesize to 128M
 
 Increase max_execution_time to 3600
 
- Let’s make the changes by executing these commands:
```
sudo sed -i "s/memory_limit = .*/memory_limit = 512M/" /etc/php/8.3/fpm/php.ini
```
```
sudo sed -i "s/upload_max_filesize = .*/upload_max_filesize = 128M/" /etc/php/8.3/fpm/php.ini
```
```
sudo sed -i "s/short_open_tag = .*/short_open_tag = On/" /etc/php/8.3/fpm/php.ini
```
```
sudo sed -i "s/max_execution_time = .*/max_execution_time = 3600/" /etc/php/8.3/fpm/php.ini
```
- Then, let’s create a PHP-FPM pool:
```
sudo vim /etc/php/8.3/fpm/pool.d/magento.conf
```
  -  We need to insert the following into the file:
  ```
  [magento]
  user = magento
  group = magento
  listen = /run/php/magento.sock
  listen.owner = magento
  listen.group = magento
  pm = ondemand
  pm.max_children = 50
  pm.start_servers = 10
  pm.min_spare_servers = 5
  pm.max_spare_servers = 10
  ```
-  Save the file and then exit from the file editor and don’t forget to restart php-fpm service:
```
sudo systemctl restart php8.3-fpm
```

### Install Nginx:
```
sudo apt install nginx -y
```
-  We need to create an nginx server block for our Magento website:
```
sudo vim /etc/nginx/sites-enabled/magento.conf
```
-  Insert the following into the configuration file:
```
upstream fastcgi_backend {
server unix:/run/php/magento.sock;
}

server {
server_name yourdomain.com;
listen 80;
set $MAGE_ROOT /opt/magento/website;
set $MAGE_MODE production;

access_log /var/log/nginx/magento-access.log;
error_log /var/log/nginx/magento-error.log;

include /opt/magento/website/nginx.conf.sample;
}
```
  - Save the file, then exit.
> [!NOTE]
>  Don’t forget to change your server name to your actual name in my case I used `localhost`.

### Install OpenSearch:
```
sudo apt install curl gnupg2
```
```
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring
```
```
echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | sudo tee /etc/apt/sources.list.d/opensearch-2.x.list
```
```
sudo apt update
```
-  With the repository information added, we can list all available versions of OpenSearch:
```
sudo apt list -a opensearch
```
- The output should be similar to this:
```hcl
magento@ip-10-101-1-245:~$ sudo apt list -a opensearch
[sudo] password for magento:
Listing... Done
opensearch/stable 2.15.0 amd64 [upgradable from: 2.11.1]
opensearch/stable 2.14.0 amd64
opensearch/stable 2.13.0 amd64
opensearch/stable 2.12.0 amd64
opensearch/stable,now 2.11.1 amd64 [installed,upgradable to: 2.15.0]
opensearch/stable 2.11.0 amd64
opensearch/stable 2.10.0 amd64
opensearch/stable 2.9.0 amd64
opensearch/stable 2.8.0 amd64
opensearch/stable 2.7.0 amd64
opensearch/stable 2.6.0 amd64
opensearch/stable 2.5.0 amd64
```
-  OpenSearch 2.11.1 by running this command below:
```
sudo apt install opensearch=2.11.1
```
-  OpenSearch uses SSL, but Magento doesn’t use it. So, we need to disable the SSL plugin in OpenSearch for successful Magento installation:
```
sudo vim /etc/opensearch/opensearch.yml
```
-  And add this to the end of yml file:
```
plugins.security.disabled: true
```
- we can enable the service and start it now:
```
sudo systemctl enable --now opensearch
```
-  Once it’s up and running, we can run this command to verify:
```
curl -X GET localhost:9200
```
  - The command will return an output similar to this:
  ```
  {
  "name" : "10.101.1.245",
  "cluster_name" : "opensearch",
  "cluster_uuid" : "zYOQTFzMQxmhhP29-u9eHA",
  "version" : {
    "distribution" : "opensearch",
    "number" : "2.11.1",
    "build_type" : "deb",
    "build_hash" : "6b1986e964d440be9137eba1413015c31c5a7752",
    "build_date" : "2023-11-29T21:43:44.221253956Z",
    "build_snapshot" : false,
    "lucene_version" : "9.7.0",
    "minimum_wire_compatibility_version" : "7.10.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "The OpenSearch Project: https://opensearch.org/"
  }
  ```

### Install MySQL Server:
```
sudo apt install mysql-server
```
```
sudo mysql
```
  ```
  mysql> CREATE USER 'magento'@'localhost' IDENTIFIED BY 'm0d1fyth15';
  mysql> CREATE DATABASE magentodb;
  mysql> GRANT ALL PRIVILEGES ON magentodb.* TO 'magento'@'localhost';
  mysql> FLUSH PRIVILEGES;
  mysql> exit
  ```

### Install Composer:
```
curl -sS https://getcomposer.org/installer -o composer-setup.php
```
```
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
```
-  To check the version of the Composer:
```
composer –version 
```
![composer](https://github.com/user-attachments/assets/86074a37-c72b-44c5-b071-ff08fcfd7f65)

### Download and Install Magento:
```
composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition=2.4.7-beta3 /opt/magento/website
```
```
cd /opt/magento/website
```
> [!IMPORTANT]
> Before running the following command, you should input the SQL database data you previously created and enter your domain:

> [!IMPORTANT]
> to create authentication keys:
> - Log in to the Commerce Marketplace https://marketplace.magento.com
> - If you don’t have an account, click Register.
> - Click your account name in the top right of the page and select My Profile.
> - Click Access Keys in the Marketplace tab.
![acesskety](https://github.com/user-attachments/assets/bd6cb4f2-421b-4ff7-9705-bf6357506de4)
> - Click Create a New Access Key. Enter a specific name for the keys and click OK.
> - New public and private keys are now associated with your account that you can click to copy. Save this information or keep the page open when working with your project. Use the Public key as your username and the Private key as your password.
> 

```

bin/magento setup:install \
--base-url=http://localhost \
--db-host=localhost \
--db-name=magentodb \
--db-user=magento \
--db-password=m0d1fyth15 \
--admin-firstname=Magento \
--admin-lastname=Admin \
--admin-email=ezzatelshazly7@gmail.com \
--admin-user=admin \
--admin-password=m0d1fyth15 \
--language=en_US \
--currency=USD \
--timezone=America/Chicago \
--use-rewrites=1 \
--search-engine=opensearch
```
-  At the end of the installation, you will see an output similar to this:
```hcl
[SUCCESS]: Magento installation complete.
[SUCCESS]: Magento Admin URI: /admin_zfc0hzf
Nothing to import.
```
![magento succes installation](https://github.com/user-attachments/assets/9aa3f704-3153-4641-9070-c66eb405c2be)

-  Before logging in to the backend, we can disable the Two Factor Authentication first and enable it again later. We need to run these commands to disable the 2FA modules:
```
php bin/magento module:disable Magento_AdminAdobeImsTwoFactorAuth
```
```
php bin/magento module:disable Magento_TwoFactorAuth
```
```
php bin/magento setup:di:compile
```
```
php bin/magento cache:clean
```
-  At this point, Magento is installed, and we can navigate to the backend at " http://yourdomain.com/admin_0ty6lcq "
-  At my case " http://localhost/admin_zfc0hzf "
> [!NOTE]
> If there is an error when you try to access Magento, ensure you replace "www-data user" with the correct user you are using. In our case, it is the "magento". The file in /etc/nginx/nginx.conf.
> 
> Also, make sure that the server name is correct in /etc/nginx/sites-enabled/magento.conf. In our case, it should be localhost.

![Magento access](https://github.com/user-attachments/assets/b05dc43e-5c85-4f21-bbdc-21e07550f7a4)

 ### Install Varnish:
 - Next, as shown in the architecture, we need to install Varnish and Redis. We also need to redirect any HTTP requests from port 80 to Varnish on port 8081.
 - This command will install the dependencies that are needed to configure the package repository:
```
sudo apt-get install debian-archive-keyring curl gnupg apt-transport-https
```
-  The next command will import the GPG key into the package manager configuration:
```
curl -s -L https://packagecloud.io/varnishcache/varnish60lts/gpgkey | sudo apt-key add –
```
-  We have to update the package list 
```
sudo apt-get update
```
-  Install Varnish by running the following command:
```
sudo apt-get install varnish
```
-  First you need to copy the original varnish.service file to the /etc/systemd/system/ folder:
```
sudo cp /lib/systemd/system/varnish.service /etc/systemd/system/
```
-  If you want to override some of the runtime parameters in the varnish.service file, you can run the following command:
```
sudo systemctl edit --full varnish
```
-  As the archeticture shown that varnish listens on port 8081 so we need to change as shown bellow:
```
ExecStart=/usr/sbin/varnishd \
	  -a :8081 \
	  -a localhost:8443,PROXY \
	  -p feature=+http2 \
	  -f /etc/varnish/default.vcl \
	  -s malloc,2g
```
-  Don’t forget to run:
```
sudo systemctl daemon-reload
```
-  The following command will replace `listen 80`; with `listen 8080`; in all virtual host files:
```
sudo find /etc/nginx -name '*.conf' -exec sed -r -i 's/\blisten ([^:]+:)?80\b([^;]*);/listen \18080\2;/g' {} ';'
```
Don’t forget to run:
```
sudo systemctl restart nginx
```
- But we need all the traffic to be redirected to Varnish on port 8081 first. Therefore, we need to modify /etc/nginx/sites-enabled/default to redirect port 80 to port 8081.
```
upstream varnish { 
server localhost:8081;
}
server{
server_name localhost;
listen localhost:80;
location  / {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_pass http://varnish;
}
}
```
-  Also change the /etc/nginx/sites-enabled/magento.conf  to listens on port 8080 as shown in the architecture:
```
}

server {
server_name localhost;
listen 8080;
set $MAGE_ROOT /opt/magento/website;
set $MAGE_MODE production;

access_log /var/log/nginx/magento-access.log;
error_log /var/log/nginx/magento-error.log;

include /opt/magento/website/nginx.conf.sample;
}
```
> [!IMPORTANT]
> When you try to access localhost now, there may be a 'bad gateway' error. Check the Varnish error log for details. The buffer size may be less than what Varnish needs, so we need to increase the buffer size in the Nginx configuration file. Add the following three lines to the http block in /etc/nginx/nginx.conf:
> 
> proxy_buffer_size   128k;
> 
> proxy_buffers   4 256k;
> 
> proxy_busy_buffers_size   256k;

### Install Redis:
```
Sudo apt-get install redis-server
```
-  Verify that Redis is running on your server  
```
systemctl status redis
```
-  PING should be the response:
```
redis-cli ping 
```
- Check the port is `6379` used for Redis:
```
netstat -plnt 
```
> [!NOTE]
> If netstat not available run this command: 
```
sudo apt install net-tools
```

### Configure Magento to use Redis for session storage:
-  Cache system settings are stored under `[site root]/app/etc/env.php`.
-  We need to edit this file. At my case it is under `/opt/magento/website/app/etc/env.php`.
```
Vim /opt/magento/website/app/etc/env.php 
```
-  And add the following:
```
'session' => [
        'save' => 'redis',
        'redis' => [
            'host' => '127.0.0.1',
            'port' => '6379',
            'timeout' => '2.5',
            'database' => '2',
            'compression_threshold' => '2048',
            'compression_library' => 'gzip',
            'log_level' => '1',
            'max_concurrency' => '6',
            'break_after_frontend' => '5',
            'break_after_adminhtml' => '30',
            'first_lifetime' => '600',
            'bot_first_lifetime' => '60',
            'bot_lifetime' => '7200',
            'disable_locking' => '0',
            'min_lifetime' => '60',
            'max_lifetime' => '2592000',
            'sentinel_master' => '',
            'sentinel_servers' => '',
            'sentinel_connect_retries' => '5',
            'sentinel_verify_master' => '0'
        ]
    ],
```
  -  You can now test to see the result.
###  The last step is to set up the SSL certificate and redirect any HTTP (port 80) requests to HTTPS (port 443), while also redirecting all traffic to Varnish as shown in the architecture.

### Generate the SSL Certificate and Key:
1. Create a directory to store the SSL certificate and key:
```
sudo mkdir -p /etc/nginx/ssl
```
2. Generate a self-signed SSL certificate:
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/self-signed.key -out /etc/nginx/ssl/self-signed.crt
```
  -  You will be prompted to enter information for the certificate. Fill in the required details as needed.	
3. Update Nginx Configuration:
  - Update /etc/nginx/sites-enabled/default
  - Modify the file to include SSL settings and redirect HTTP to HTTPS:
  ```
upstream varnish {
    server localhost:8081;
}

server {
    server_name localhost;
    listen 80;
    listen [::]:80;
    return 301 https://$server_name$request_uri;
}

server {
    server_name localhost;
    listen 443 ssl;
    listen [::]:443 ssl;

    ssl_certificate /etc/nginx/ssl/self-signed.crt;
    ssl_certificate_key /etc/nginx/ssl/self-signed.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_pass http://varnish;
    }
}
  ```

4. Update /etc/nginx/sites-enabled/magento.conf:
  -  Modify the file to include SSL settings and redirect HTTP to HTTPS:
  ```
upstream fastcgi_backend {
  server unix:/run/php/magento.sock;
}

server {
    server_name localhost;
    listen 8080;
    set $MAGE_ROOT /opt/magento/website;
    set $MAGE_MODE production;

    access_log /var/log/nginx/magento-access.log;
    error_log /var/log/nginx/magento-error.log;

    include /opt/magento/website/nginx.conf.sample;
}

server {
    server_name localhost;
    listen 80;
    listen [::]:80;
    return 301 https://$server_name$request_uri;
}

server {
    server_name localhost;
    listen 443 ssl;
    listen [::]:443 ssl;

    ssl_certificate /etc/nginx/ssl/self-signed.crt;
    ssl_certificate_key /etc/nginx/ssl/self-signed.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    set $MAGE_ROOT /opt/magento/website;
    set $MAGE_MODE production;

    access_log /var/log/nginx/magento-ssl-access.log;
    error_log /var/log/nginx/magento-ssl-error.log;

    include /opt/magento/website/nginx.conf.sample;
}

  ```

5. Test the Nginx configuration:
```
sudo nginx -t
```
6. Reload Nginx to apply the changes:
```
sudo systemctl reload nginx
```
![redirect to https with ssl](https://github.com/user-attachments/assets/9a75167a-8155-4ab7-8bcb-6ff137385580)

![https devolper mode](https://github.com/user-attachments/assets/0fefca8f-b55d-4d5f-bf6f-8003c4b2d442)
