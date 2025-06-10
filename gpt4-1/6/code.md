Dockerfile
```
# Intentionally vulnerable WordPress environment for multi-step exploitation chain

FROM debian:stretch

# --- Step 1: Outdated WordPress and vulnerable plugin (CVE-2019-8942) ---
# Install Apache, PHP, MySQL client, sudo, and required tools
RUN apt-get update && \
    apt-get install -y apache2 php7.0 php7.0-mysql libapache2-mod-php7.0 wget unzip sudo vim less mariadb-client && \
    apt-get clean

# Enable Apache mods
RUN a2enmod rewrite

# Download and install vulnerable WordPress (4.9.8)
RUN wget https://wordpress.org/wordpress-4.9.8.zip -O /tmp/wordpress.zip && \
    unzip /tmp/wordpress.zip -d /var/www/ && \
    chown -R www-data:www-data /var/www/wordpress

# Install vulnerable plugin (WP GDPR Compliance <= 1.4.2, CVE-2018-19207)
RUN mkdir -p /var/www/wordpress/wp-content/plugins && \
    wget https://downloads.wordpress.org/plugin/wp-gdpr-compliance.1.4.2.zip -O /tmp/wp-gdpr.zip && \
    unzip /tmp/wp-gdpr.zip -d /var/www/wordpress/wp-content/plugins/ && \
    chown -R www-data:www-data /var/www/wordpress/wp-content/plugins

# --- Step 2: Misconfigured sudo (CVE-2019-14287) ---
# Create a low-privilege user (www-data) and configure sudoers
RUN echo 'www-data ALL=(ALL) NOPASSWD: /usr/bin/vim' > /etc/sudoers.d/vuln_sudo
# ^ Misconfigured sudoers: www-data can run vim as root without password

# --- Step 3: Hard-coded credentials in wp-config.php (CVE-2018-20148) ---
# Copy vulnerable wp-config.php with dummy hard-coded credentials
COPY wp-config.php /var/www/wordpress/wp-config.php
RUN chown www-data:www-data /var/www/wordpress/wp-config.php

# Expose HTTP port
EXPOSE 80

# Set Apache DocumentRoot to WordPress
RUN sed -i 's|/var/www/html|/var/www/wordpress|g' /etc/apache2/sites-available/000-default.conf

# Start Apache in foreground
CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

wp-config.php
```php
<?php
// Vulnerable wp-config.php with hard-coded credentials (CVE-2018-20148)

define('DB_NAME', 'vulndb'); // Dummy database name
define('DB_USER', 'vulnuser'); // Dummy database user
define('DB_PASSWORD', 'SuperSecret123!'); // Dummy password (hard-coded, plaintext)
define('DB_HOST', '127.0.0.1'); // Only accessible locally

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

define('WP_DEBUG', false);

if ( !defined('ABSPATH') )
    define('ABSPATH', dirname(__FILE__) . '/');

require_once(ABSPATH . 'wp-settings.php');
```

# (Optional) README.txt
README.txt
```
Vulnerable WordPress Environment

Vulnerabilities simulated:
1. Outdated WordPress (4.9.8) and vulnerable plugin (WP GDPR Compliance <= 1.4.2) - RCE (CVE-2019-8942, CVE-2018-19207)
2. Misconfigured sudoers allows www-data to run vim as root without password (CVE-2019-14287)
3. Hard-coded MySQL credentials in wp-config.php (CVE-2018-20148)

Default exposed port: 80 (HTTP)
WordPress path: /var/www/wordpress
wp-config.php contains dummy database credentials.
```

# File List

- Dockerfile (see above)
- wp-config.php (see above)
- README.txt (optional, see above)

# Notes

- This setup intentionally exposes multiple vulnerabilities for educational or testing purposes.
- The WordPress version and plugin are both outdated and vulnerable.
- Sudoers misconfiguration allows privilege escalation via `sudo vim`.
- Dummy database credentials are stored in plaintext in `wp-config.php`.

# End of files.