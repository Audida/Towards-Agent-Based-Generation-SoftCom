Dockerfile
```Dockerfile
# Dockerfile
# Vulnerable Environment: Outdated WordPress (CVE-2019-8942), Misconfigured Sudo (CVE-2019-14287), Hardcoded DB creds (CVE-2018-20148)

FROM debian:9

# --- Step 1: Install vulnerable WordPress and dependencies ---
RUN apt-get update && \
    apt-get install -y apache2 php7.0 php7.0-mysql libapache2-mod-php7.0 wget unzip mariadb-server sudo && \
    apt-get clean

# Download WordPress 5.0 (vulnerable to CVE-2019-8942)
RUN wget https://wordpress.org/wordpress-5.0.zip -O /tmp/wordpress.zip && \
    unzip /tmp/wordpress.zip -d /tmp && \
    mv /tmp/wordpress/* /var/www/html/ && \
    chown -R www-data:www-data /var/www/html

# --- Step 2: Setup MySQL with hardcoded credentials (CVE-2018-20148) ---
COPY db-init.sql /db-init.sql
RUN service mysql start && \
    mysql -u root -e "CREATE DATABASE wordpress;" && \
    mysql -u root wordpress < /db-init.sql

# --- Step 3: Insert vulnerable wp-config.php with hardcoded creds ---
COPY wp-config.php /var/www/html/wp-config.php
RUN chown www-data:www-data /var/www/html/wp-config.php

# --- Step 4: Setup misconfigured sudo and writable backup script (CVE-2019-14287) ---
# Create a backup script owned by root but writable by www-data
COPY backup.sh /usr/local/bin/backup.sh
RUN chmod 775 /usr/local/bin/backup.sh && \
    chown root:www-data /usr/local/bin/backup.sh

# Add www-data to sudoers with NOPASSWD for backup.sh (overly broad)
COPY sudoers.d_www-data /etc/sudoers.d/www-data
RUN chmod 440 /etc/sudoers.d/www-data

# --- Step 5: Expose HTTP port and start services ---
EXPOSE 80

# Start all services (Apache, MySQL) in foreground
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

CMD ["/entrypoint.sh"]
```

db-init.sql
```sql
-- db-init.sql
-- Dummy WordPress DB content for demonstration
CREATE TABLE IF NOT EXISTS wp_users (
  ID bigint(20) unsigned NOT NULL AUTO_INCREMENT,
  user_login varchar(60) NOT NULL DEFAULT '',
  user_pass varchar(255) NOT NULL DEFAULT '',
  user_email varchar(100) NOT NULL DEFAULT '',
  PRIMARY KEY (ID)
);

INSERT INTO wp_users (user_login, user_pass, user_email) VALUES
('admin', '$P$B5u5QJYw5Vv5YQw5Vv5YQw5Vv5YQw5/', 'admin@example.com');
```

wp-config.php
```php
<?php
// wp-config.php
// Hardcoded DB credentials (CVE-2018-20148)
define('DB_NAME', 'wordpress');
define('DB_USER', 'root');
define('DB_PASSWORD', 'vulnerablepassword'); // Dummy password
define('DB_HOST', 'localhost');
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', '');

// Dummy keys for demonstration
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

backup.sh
```bash
#!/bin/bash
# /usr/local/bin/backup.sh
# Writable backup script, executable via sudo by www-data (CVE-2019-14287)
# Vulnerability: www-data can edit this script and execute as root

tar czf /tmp/backup.tar.gz /var/www/html
echo "Backup completed."
```

sudoers.d_www-data
```text
# /etc/sudoers.d/www-data
# Overly permissive sudoers entry (CVE-2019-14287)
www-data ALL=(ALL:ALL) NOPASSWD: /usr/local/bin/backup.sh
```

entrypoint.sh
```bash
#!/bin/bash
# entrypoint.sh
# Starts MySQL and Apache in foreground for Docker

service mysql start
service apache2 start

# Keep container running
tail -f /dev/null
```

# File List

- Dockerfile
- db-init.sql
- wp-config.php
- backup.sh
- sudoers.d_www-data
- entrypoint.sh

# Vulnerabilities Simulated

1. **Outdated WordPress (CVE-2019-8942):** WordPress 5.0 with file upload RCE, exposed on port 80.
2. **Misconfigured Sudo (CVE-2019-14287):** www-data can sudo a writable script as root without password.
3. **Hardcoded DB Credentials (CVE-2018-20148):** wp-config.php contains plaintext root MySQL credentials.

This setup enables a single-container vulnerable environment for the described attack chain.