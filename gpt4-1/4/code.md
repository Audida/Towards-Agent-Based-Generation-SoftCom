Dockerfile
```
# Dockerfile
# Vulnerable Environment: Outdated Drupal (CVE-2019-6340), Alpine with blank root password (CVE-2019-5021), Misconfigured Redis (CVE-2015-4335)

FROM alpine:3.7

# --- Step 2: CVE-2019-5021 - Alpine with blank root password ---
# This Alpine version ships with a blank root password by default.

# Install required packages: PHP, Apache, Drupal dependencies, Redis
RUN apk update && \
    apk add --no-cache \
        apache2 \
        php7 \
        php7-apache2 \
        php7-gd \
        php7-pdo \
        php7-pdo_mysql \
        php7-mysqli \
        php7-json \
        php7-session \
        php7-xml \
        php7-simplexml \
        php7-tokenizer \
        php7-ctype \
        php7-curl \
        php7-dom \
        php7-mbstring \
        php7-zip \
        php7-openssl \
        php7-phar \
        php7-iconv \
        php7-fileinfo \
        php7-intl \
        php7-pdo_pgsql \
        php7-pgsql \
        wget \
        curl \
        supervisor \
        redis

# --- Step 1: CVE-2019-6340 - Outdated Drupal with RCE ---
# Download and extract vulnerable Drupal version (8.6.9)
WORKDIR /var/www/localhost/htdocs
RUN wget https://ftp.drupal.org/files/projects/drupal-8.6.9.tar.gz && \
    tar -xzf drupal-8.6.9.tar.gz --strip-components=1 && \
    rm drupal-8.6.9.tar.gz

# --- Step 2: Hard-coded credentials in settings.php ---
# Copy vulnerable settings.php with hard-coded DB credentials
COPY settings.php sites/default/settings.php

# --- Step 3: Misconfigured Redis (CVE-2015-4335) ---
# Redis will be started with default config (no auth, listens on all interfaces)

# --- Apache config: Expose HTTP on port 80 ---
RUN sed -i 's/Listen 80/Listen 0.0.0.0:80/' /etc/apache2/httpd.conf

# --- Supervisor config to run both Apache and Redis ---
COPY supervisord.conf /etc/supervisord.conf

# --- Permissions for Drupal ---
RUN chown -R apache:apache /var/www/localhost/htdocs/sites

# --- Expose ports ---
EXPOSE 80    # Apache (Drupal)
EXPOSE 6379  # Redis

# --- Start supervisor (runs Apache and Redis) ---
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
```

settings.php
```php
<?php
// sites/default/settings.php
// --- Step 2: Hard-coded credentials vulnerability ---
// Dummy DB credentials reused for OS user as well (bad practice)
$databases['default']['default'] = array (
  'database' => 'drupal',
  'username' => 'drupaluser',
  'password' => 'SuperSecret123', // <-- Hard-coded credential
  'host' => 'localhost',
  'driver' => 'mysql',
  'prefix' => '',
  'port' => '',
  'namespace' => 'Drupal\\Core\\Database\\Driver\\mysql',
  'autoload' => 'core/modules/system/src/Database/Driver/mysql/',
);

// Dummy hash salt
$settings['hash_salt'] = 'vulnerable_dummy_salt';

// Enable RESTful web services (required for CVE-2019-6340 exploitation)
$settings['rest'] = TRUE;

// Other insecure settings for demonstration
$settings['trusted_host_patterns'] = array('.*');
```

supervisord.conf
```
; /etc/supervisord.conf
; Supervisor config to run both Apache and Redis

[supervisord]
nodaemon=true

[program:httpd]
command=/usr/sbin/httpd -DFOREGROUND
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stderr_logfile=/dev/stderr

[program:redis]
command=/usr/bin/redis-server --protected-mode no --bind 0.0.0.0
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stderr_logfile=/dev/stderr
```

# File List

- Dockerfile
- sites/default/settings.php
- /etc/supervisord.conf

# Directory Structure

```
.
├── Dockerfile
├── supervisord.conf
└── sites
    └── default
        └── settings.php
```

# Key Vulnerabilities Implemented

- **Step 1: Outdated Drupal (8.6.9) with REST enabled** — vulnerable to CVE-2019-6340 (RCE via HTTP POST).
- **Step 2: Hard-coded credentials in settings.php** — DB credentials (`drupaluser:SuperSecret123`) reused for OS user, and Alpine 3.7 with blank root password (CVE-2019-5021).
- **Step 3: Redis runs with default config** — no authentication, listens on all interfaces (CVE-2015-4335), accessible from within the container.

# Usage Notes

- Build context should include the `sites/default/settings.php` file and `supervisord.conf` at the root.
- On container start, both Apache (serving Drupal) and Redis (misconfigured) will run.
- The environment is intentionally insecure for demonstration and training purposes only.