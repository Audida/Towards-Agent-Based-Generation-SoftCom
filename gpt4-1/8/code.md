Dockerfile
```Dockerfile
# Dockerfile
# Vulnerable Environment: Outdated WordPress (CVE-2019-8942), hard-coded DB creds (CVE-2019-5420), misconfigured SSH (CVE-2018-10933)

FROM ubuntu:18.04

# Install Apache, PHP, MySQL, SSH, and dependencies
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
    apache2 \
    php7.2 php7.2-mysql libapache2-mod-php7.2 \
    wget unzip \
    mysql-server \
    openssh-server \
    libssh-0 libssh-dev \
    supervisor && \
    apt-get clean

# --- Step 1: Outdated WordPress (CVE-2019-8942) ---
# Download and install WordPress 5.0 (vulnerable version)
RUN wget https://wordpress.org/wordpress-5.0.zip -O /tmp/wordpress.zip && \
    unzip /tmp/wordpress.zip -d /var/www/html/ && \
    chown -R www-data:www-data /var/www/html/wordpress

# --- Step 2: Hard-coded DB credentials (CVE-2019-5420) ---
# Copy vulnerable wp-config.php with hard-coded credentials
COPY wp-config.php /var/www/html/wordpress/wp-config.php

# --- Step 3: Misconfigured SSH (CVE-2018-10933) ---
# Create a dummy internal user with known credentials
RUN useradd -m internaluser && echo 'internaluser:internalpass' | chpasswd

# Configure SSH to allow password authentication and listen on a custom port
RUN mkdir /var/run/sshd
COPY sshd_config /etc/ssh/sshd_config

# Expose HTTP and SSH ports
EXPOSE 80 2222

# --- MySQL Setup ---
# Set MySQL root password and create DB/user for WordPress
RUN service mysql start && \
    mysql -e "CREATE DATABASE wordpress;" && \
    mysql -e \"CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'wppassword';\" && \
    mysql -e \"GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';\" && \
    mysql -e \"FLUSH PRIVILEGES;\" && \
    mysql -u root -e \"CREATE TABLE wordpress.internal_creds (id INT PRIMARY KEY AUTO_INCREMENT, username VARCHAR(50), password VARCHAR(50));\" wordpress && \
    mysql -u root -e \"INSERT INTO wordpress.internal_creds (username, password) VALUES ('internaluser', 'internalpass');\" wordpress

# Supervisor config to run all services
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Start all services
CMD ["/usr/bin/supervisord"]

```

wp-config.php
```php
<?php
// wp-config.php (with hard-coded credentials) - CVE-2019-5420
define('DB_NAME', 'wordpress');
define('DB_USER', 'wpuser');
define('DB_PASSWORD', 'wppassword'); // Hard-coded credentials
define('DB_HOST', 'localhost');
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', '');

$table_prefix  = 'wp_';

define('AUTH_KEY',         'dummykey1');
define('SECURE_AUTH_KEY',  'dummykey2');
define('LOGGED_IN_KEY',    'dummykey3');
define('NONCE_KEY',        'dummykey4');
define('AUTH_SALT',        'dummysalt1');
define('SECURE_AUTH_SALT', 'dummysalt2');
define('LOGGED_IN_SALT',   'dummysalt3');
define('NONCE_SALT',       'dummysalt4');

define('FS_METHOD', 'direct');

if ( !defined('ABSPATH') )
    define('ABSPATH', dirname(__FILE__) . '/');

require_once(ABSPATH . 'wp-settings.php');
```

sshd_config
```text
# sshd_config - Misconfigured SSH (CVE-2018-10933)
Port 2222
PermitRootLogin yes
PasswordAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
AllowUsers internaluser
# WARNING: No restrictions on source IP, allows lateral movement from web server
```

supervisord.conf
```ini
[supervisord]
nodaemon=true

[program:apache2]
command=/usr/sbin/apachectl -DFOREGROUND

[program:sshd]
command=/usr/sbin/sshd -D

[program:mysql]
command=/usr/bin/mysqld_safe
```

# File List

- Dockerfile
- wp-config.php
- sshd_config
- supervisord.conf

# Notes on Vulnerabilities

- **Step 1:** WordPress 5.0 (CVE-2019-8942) is installed, allowing RCE via file upload.
- **Step 2:** wp-config.php contains hard-coded MySQL credentials (CVE-2019-5420).
- **Step 3:** SSH is misconfigured to allow password authentication for a known user on port 2222 (CVE-2018-10933 pattern).

This setup provides a realistic, intentionally vulnerable environment for simulating the described attack chain.