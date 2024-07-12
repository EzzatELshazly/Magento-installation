# Magento-installation
# Create a new system user
/usr/sbin/adduser \
   --system \
   --shell /bin/bash \
   --gecos 'Magento user' \
   --group \
   --home /opt/magento \
magento

# Set password for the new user (interactive)
passwd magento

# Give the new user sudo privilege
usermod -aG sudo magento

# Switch to the new user
su - magento

# Install PHP 8.3 and its extensions
sudo apt install php-{bcmath,common,curl,fpm,gd,intl,mbstring,mysql,soap,xml,xsl,zip,cli}

# Modify PHP settings
sudo sed -i "s/memory_limit = .*/memory_limit = 512M/" /etc/php/8.3/fpm/php.ini
sudo sed -i "s/upload_max_filesize = .*/upload_max_filesize = 128M/" /etc/php/8.3/fpm/php.ini
sudo sed -i "s/short_open_tag = .*/short_open_tag = On/" /etc/php/8.3/fpm/php.ini
sudo sed -i "s/max_execution_time = .*/max_execution_time = 3600/" /etc/php/8.3/fpm/php.ini

# Create a PHP-FPM pool
sudo bash -c 'cat <<EOF > /etc/php/8.3/fpm/pool.d/magento.conf
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
EOF'

# Restart PHP-FPM service
sudo systemctl restart php8.3-fpm

# Install Nginx
sudo apt install nginx -y

# Create an Nginx server block for Magento
sudo bash -c 'cat <<EOF > /etc/nginx/sites-enabled/magento.conf
upstream fastcgi_backend {
    server unix:/run/php/magento.sock;
}

server {
    server_name localhost;
    listen 80;
    set \$MAGE_ROOT /opt/magento/website;
    set \$MAGE_MODE production;

    access_log /var/log/nginx/magento-access.log;
    error_log /var/log/nginx/magento-error.log;

    include /opt/magento/website/nginx.conf.sample;
}
EOF'

# Install OpenSearch
sudo apt install curl gnupg2 -y
curl -o- https://artifacts.opensearch.org/publickeys/opensearch.pgp | sudo gpg --dearmor --batch --yes -o /usr/share/keyrings/opensearch-keyring
echo "deb [signed-by=/usr/share/keyrings/opensearch-keyring] https://artifacts.opensearch.org/releases/bundle/opensearch/2.x/apt stable main" | sudo tee /etc/apt/sources.list.d/opensearch-2.x.list
sudo apt update
sudo apt install opensearch=2.11.1 -y

# Disable SSL plugin in OpenSearch
sudo bash -c 'echo "plugins.security.disabled: true" >> /etc/opensearch/opensearch.yml'

# Enable and start OpenSearch service
sudo systemctl enable --now opensearch

# Verify OpenSearch installation
curl -X GET localhost:9200

# Install MySQL Server
sudo apt install mysql-server -y
sudo mysql -e "CREATE USER 'magento'@'localhost' IDENTIFIED BY 'm0d1fyth15';"
sudo mysql -e "CREATE DATABASE magentodb;"
sudo mysql -e "GRANT ALL PRIVILEGES ON magentodb.* TO 'magento'@'localhost';"
sudo mysql -e "FLUSH PRIVILEGES;"

# Install Composer
curl -sS https://getcomposer.org/installer -o composer-setup.php
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer

# Check Composer version
composer --version

# Download and install Magento
composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition=2.4.7-beta3 /opt/magento/website
cd /opt/magento/website

# Install Magento
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

# Disable Two Factor Authentication
php bin/magento module:disable Magento_AdminAdobeImsTwoFactorAuth
php bin/magento module:disable Magento_TwoFactorAuth
php bin/magento setup:di:compile
php bin/magento cache:clean

# Install Varnish
sudo apt-get install debian-archive-keyring curl gnupg apt-transport-https -y
curl -s -L https://packagecloud.io/varnishcache/varnish60lts/gpgkey | sudo apt-key add -
sudo apt-get update
sudo apt-get install varnish -y

# Copy Varnish service file
sudo cp /lib/systemd/system/varnish.service /etc/systemd/system/

# Edit Varnish service file to listen on port 8081
sudo bash -c 'cat <<EOF > /etc/systemd/system/varnish.service
ExecStart=/usr/sbin/varnishd \\
    -a :8081 \\
    -a localhost:8443,PROXY \\
    -p feature=+http2 \\
    -f /etc/varnish/default.vcl \\
    -s malloc,2g
EOF'

# Reload systemd daemon
sudo systemctl daemon-reload

# Modify Nginx configuration to listen on port 8080
sudo find /etc/nginx -name '*.conf' -exec sed -r -i 's/\blisten ([^:]+:)?80\b([^;]*);/listen \18080\2;/g' {} ';'
sudo systemctl restart nginx

# Redirect traffic from port 80 to Varnish on port 8081
sudo bash -c 'cat <<EOF > /etc/nginx/sites-enabled/default
upstream varnish { 
    server localhost:8081;
}

server {
    server_name localhost;
    listen localhost:80;

    location / {
        proxy_set_header Host \$host;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_pass http://varnish;
    }
}
EOF'

# Update Nginx configuration to listen on port 8080 for Magento
sudo bash -c 'cat <<EOF > /etc/nginx/sites-enabled/magento.conf
upstream fastcgi_backend {
    server unix:/run/php/magento.sock;
}

server {
    server_name localhost;
    listen 8080;
    set \$MAGE_ROOT /opt/magento/website;
    set \$MAGE_MODE production;

    access_log /var/log/nginx/magento-access.log;
    error_log /var/log/nginx/magento-error.log;

    include /opt/magento/website/nginx.conf.sample;
}
EOF'

# Increase buffer size in Nginx configuration
sudo bash -c 'cat <<EOF >> /etc/nginx/nginx.conf
proxy_buffer_size   128k;
proxy_buffers   4 256k;
proxy_busy_buffers_size   256k;
EOF'

# Install Redis
sudo apt-get install redis-server -y

# Verify Redis installation
service redis status
redis-cli ping

# Configure Magento to use Redis for session storage
sudo bash -c 'cat <<EOF >> /opt/magento/website/app/etc/env.php
"session" => [
    "save" => "redis",
    "redis" => [
        "host" => "127.0.0.1",
        "port" => "6379",
        "timeout" => "2.5",
        "database" => "2",
        "compression_threshold" => "2048",
        "compression_library" => "gzip",
        "log_level" => "1",
        "max_concurrency" => "6",
        "break_after_frontend" => "5",
        "break_after_adminhtml" => "30",
        "first_lifetime" => "600",
        "bot_first_lifetime" => "60",
        "bot_lifetime" => "7200",
        "disable_locking" => "0",
        "min_lifetime" => "60",
        "max_lifetime" => "2592000",
        "sentinel_master" => "",
        "sentinel_servers" => "",
        "sentinel_connect_retries" => "5",
        "sentinel_verify_master" => "0"
    ]
],
EOF'

# Setup SSL Certificate
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/magento.key -out /etc/nginx/ssl/magento.crt

# Modify Nginx configuration for SSL
sudo bash -c 'cat <<EOF >> /etc/nginx/sites-enabled/magento.conf
server {
    listen 443 ssl;
    server_name localhost;
    ssl_certificate /etc/nginx/ssl/magento.crt;
    ssl_certificate_key /etc/nginx/ssl/magento.key;

    set \$MAGE_ROOT /opt/magento/website;
    set \$MAGE_MODE production;

    access_log /var/log/nginx/magento-ssl-access.log;
    error_log /var/log/nginx/magento-ssl-error.log;

    include /opt/magento/website/nginx.conf.sample;
}
EOF'

# Restart Nginx service
sudo systemctl restart nginx
