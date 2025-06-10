Dockerfile
```Dockerfile
# Use an old PHP+Apache base image to simulate outdated environment
FROM php:7.2-apache

# --- Step 1: Outdated WordPress with RCE (CVE-2019-8942) ---
# Download and extract WordPress 4.9.8 (vulnerable to CVE-2019-8942)
RUN apt-get update && \
    apt-get install -y wget unzip mariadb-client && \
    wget https://wordpress.org/wordpress-4.9.8.zip -O /tmp/wordpress.zip && \
    unzip /tmp/wordpress.zip -d /var/www/html/ && \
    cp -r /var/www/html/wordpress/* /var/www/html/ && \
    rm -rf /var/www/html/wordpress /tmp/wordpress.zip

# --- Step 2: Hard-coded MySQL credentials in wp-config.php (CVE-2018-20148) ---
# Copy our intentionally vulnerable wp-config.php with dummy credentials
COPY wp-config.php /var/www/html/wp-config.php

# --- Step 3: Misconfigured Docker socket (CVE-2019-5736) ---
# Create a world-writable Docker socket mount point (simulated for demo)
RUN mkdir -p /var/run && \
    touch /var/run/docker.sock && \
    chmod 666 /var/run/docker.sock
# Note: In a real setup, the host's docker.sock would be mounted with -v

# --- Additional Insecure Configurations ---
# Expose HTTP port
EXPOSE 80

# Set weak file permissions for demonstration (world-writable web root)
RUN chmod -R 777 /var/www/html

# Dummy entrypoint (Apache)
CMD ["apache2-foreground"]
```

wp-config.php
```php
<?php
/**
 * Vulnerable wp-config.php for demonstration
 * - Hard-coded MySQL credentials (CVE-2018-20148)
 * - World-readable file permissions (for demo)
 */

// ** MySQL settings - You can get this info from your web host ** //
define('DB_NAME', 'vulndb'); // Dummy database name
define('DB_USER', 'vulnuser'); // Dummy username
define('DB_PASSWORD', 'SuperSecret123!'); // Dummy password (hard-coded, weak)
define('DB_HOST', 'localhost');
define('DB_CHARSET', 'utf8');
define('DB_COLLATE', '');

// ** Authentication Unique Keys and Salts. (Dummy values) **
define('AUTH_KEY',         'dummykey1');
define('SECURE_AUTH_KEY',  'dummykey2');
define('LOGGED_IN_KEY',    'dummykey3');
define('NONCE_KEY',        'dummykey4');
define('AUTH_SALT',        'dummysalt1');
define('SECURE_AUTH_SALT', 'dummysalt2');
define('LOGGED_IN_SALT',   'dummysalt3');
define('NONCE_SALT',       'dummysalt4');

/**#@-*/

/**
 * WordPress Database Table prefix.
 */
$table_prefix  = 'wp_';

/**
 * For developers: WordPress debugging mode.
 */
define('WP_DEBUG', false);

/* That's all, stop editing! Happy publishing. */

/** Absolute path to the WordPress directory. */
if ( !defined('ABSPATH') )
	define('ABSPATH', dirname(__FILE__) . '/');

/** Sets up WordPress vars and included files. */
require_once(ABSPATH . 'wp-settings.php');
?>
```

# Directory structure:
```
.
├── Dockerfile
└── wp-config.php
```

# Notes on Vulnerabilities Simulated:
- **Step 1 (CVE-2019-8942):** WordPress 4.9.8 is vulnerable to RCE via media upload.
- **Step 2 (CVE-2018-20148):** wp-config.php contains hard-coded, plaintext MySQL credentials.
- **Step 3 (CVE-2019-5736):** /var/run/docker.sock is present and world-writable, simulating a misconfigured Docker socket for lateral movement.

# Usage:
Build and run the container (for demonstration only; do NOT expose to the internet):
```sh
docker build -t vuln-wp-demo .
docker run -d -p 8080:80 -v /var/run/docker.sock:/var/run/docker.sock vuln-wp-demo
```
(If you want to simulate the real Docker socket exposure, use the `-v` flag as above.)

This setup provides a realistic, intentionally vulnerable environment for demonstrating the described attack chain.