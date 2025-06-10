Dockerfile:
```bash
# Use an outdated Alpine Linux image
FROM alpine:3.12

# Install Apache, PHP, and Drupal 7
RUN apk add --no-cache apache2 php php-fpm php-mysql php-xml php-zip php-gd php-curl php-mbstring php-json drupal7

# Copy Drupal 7 files
COPY drupal7 /var/www/html

# Set Apache to run PHP as FPM
RUN echo 'Action php-fpm /usr/bin/php-fpm' >> /etc/apache2/conf-available/php7.conf
RUN ln -s /etc/apache2/conf-available/php7.conf /etc/apache2/conf-enabled/

# Expose port 80
EXPOSE 80

# Run Apache and PHP-FPM on container start
CMD ["apache2-foreground", "php-fpm"]
```

drupal7/sites/default/default.settings.php:
```php
<?php
$databases = array (
  'default' => array (
    'default' => array (
      'database' => 'db_name',
      'username' => 'db_user',
      'password' => 'db_password',
      'host' => 'db_host',
      'port' => 'db_port',
      'driver' => 'mysql',
    ),
  ),
);
```

drupal7/sites/default/files/.search_index_files:
```bash
db_name:db_user:db_password
```

Docker Compose file (docker-compose.yml):
```yaml
version: '3'
services:
  web:
    build: .
    ports:
      - "80:80"
```

This Docker setup includes an outdated Alpine Linux image with Apache, PHP, and Drupal 7 installed. The Drupal 7 instance contains the Drupalgeddon2 vulnerability (CVE-2018-7600) and the hard-coded database credentials vulnerability (CVE-2017-6074). The Docker Compose file sets up port 80 for external access. The attacker can exploit the Drupalgeddon2 vulnerability to gain shell access on the web server and access the sensitive data stored in the database.