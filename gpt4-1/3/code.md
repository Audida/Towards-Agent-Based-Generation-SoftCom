Dockerfile
```
# Intentionally vulnerable environment simulating:
# 1. Outdated WordPress (CVE-2019-8942, RCE)
# 2. Hard-coded credentials in config (CVE-2018-1000525)
# 3. Misconfigured Samba share (CVE-2017-7494)

FROM ubuntu:18.04

# Install dependencies
RUN apt-get update && \
    apt-get install -y apache2 php php-mysql libapache2-mod-php wget unzip mariadb-server samba && \
    apt-get clean

# --- Step 1: Outdated WordPress (CVE-2019-8942) ---
WORKDIR /var/www/html
RUN wget https://wordpress.org/wordpress-5.0.zip && \
    unzip wordpress-5.0.zip && \
    mv wordpress/* . && \
    rm -rf wordpress wordpress-5.0.zip

# --- Step 2: Hard-coded credentials in config (CVE-2018-1000525) ---
# Copy vulnerable wp-config.php with hard-coded credentials
COPY wp-config.php /var/www/html/wp-config.php

# --- Step 3: Misconfigured Samba share (CVE-2017-7494) ---
# Copy vulnerable smb.conf
COPY smb.conf /etc/samba/smb.conf

# Create a shared directory with sensitive files
RUN mkdir -p /srv/share && \
    echo "Sensitive Employee Data: SSN: 123-45-6789" > /srv/share/employee_records.txt && \
    echo "Proprietary Info: SECRET_PROJECT_XYZ" > /srv/share/proprietary_info.txt && \
    chmod -R 0777 /srv/share

# Set up database with same credentials as in wp-config.php
RUN service mysql start && \
    mysql -e "CREATE DATABASE wordpress;" && \
    mysql -e "CREATE USER 'vulnuser'@'localhost' IDENTIFIED BY 'SuperSecretPass123';" && \
    mysql -e "GRANT ALL PRIVILEGES ON wordpress.* TO 'vulnuser'@'localhost';" && \
    mysql -e "FLUSH PRIVILEGES;"

# Set up a local user with same credentials as DB (for privilege escalation)
RUN useradd -m vulnuser && echo 'vulnuser:SuperSecretPass123' | chpasswd

# Expose HTTP and SMB ports
EXPOSE 80 445

# Start all services (insecurely, for demo)
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

CMD ["/entrypoint.sh"]
```

wp-config.php
```php
<?php
// Vulnerable wp-config.php with hard-coded credentials (CVE-2018-1000525)

// ** Database settings ** //
define('DB_NAME', 'wordpress');
define('DB_USER', 'vulnuser'); // Hard-coded credential
define('DB_PASSWORD', 'SuperSecretPass123'); // Hard-coded credential
define('DB_HOST', 'localhost');
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', '');

// ** Authentication Unique Keys and Salts. **
define('AUTH_KEY',         'dummykey1');
define('SECURE_AUTH_KEY',  'dummykey2');
define('LOGGED_IN_KEY',    'dummykey3');
define('NONCE_KEY',        'dummykey4');
define('AUTH_SALT',        'dummykey5');
define('SECURE_AUTH_SALT', 'dummykey6');
define('LOGGED_IN_SALT',   'dummykey7');
define('NONCE_SALT',       'dummykey8');

$table_prefix  = 'wp_';

define('WP_DEBUG', false);

if ( !defined('ABSPATH') )
    define('ABSPATH', dirname(__FILE__) . '/');

require_once(ABSPATH . 'wp-settings.php');
?>
```

smb.conf
```
# Vulnerable Samba config (CVE-2017-7494)
[global]
   workgroup = WORKGROUP
   server string = Samba Server
   log file = /var/log/samba/log.%m
   max log size = 50
   security = user
   map to guest = Bad User
   passdb backend = tdbsam

[share]
   path = /srv/share
   browsable = yes
   writable = yes
   guest ok = yes
   read only = no
   public = yes
   # No restrictions: Any user can write/upload (enables exploitation)
```

entrypoint.sh
```bash
#!/bin/bash
# Start all services insecurely for demonstration

service mysql start
service apache2 start
service smbd start

# Keep container running
tail -f /dev/null
```

# File List:
- Dockerfile
- wp-config.php
- smb.conf
- entrypoint.sh

# Notes:
- WordPress 5.0 (vulnerable to CVE-2019-8942) is installed.
- wp-config.php contains hard-coded credentials reused for both DB and local user (CVE-2018-1000525).
- Samba is configured with a world-writable share and no access controls (CVE-2017-7494).
- Dummy sensitive files are present in the share.
- All services are started in a single container for easy exploitation chain simulation.

This setup provides a realistic, intentionally vulnerable environment for demonstrating the described attack chain.