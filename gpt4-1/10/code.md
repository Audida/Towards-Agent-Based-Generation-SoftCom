Dockerfile
```Dockerfile
# Dockerfile
# Intentionally vulnerable environment simulating a Drupal RCE (CVE-2019-6340),
# plaintext config credentials (CVE-2018-15133), and credential reuse for SSH (CVE-2016-6563).

FROM php:7.2-apache

# Step 1: Install vulnerable Drupal version (8.6.9, vulnerable to CVE-2019-6340)
RUN apt-get update && \
    apt-get install -y wget unzip mariadb-server openssh-server && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /var/www/html

# Download and extract vulnerable Drupal
RUN wget https://ftp.drupal.org/files/projects/drupal-8.6.9.zip && \
    unzip drupal-8.6.9.zip && \
    mv drupal-8.6.9/* . && \
    rm -rf drupal-8.6.9 drupal-8.6.9.zip

# Step 2: Add a vulnerable settings.php with plaintext credentials (CVE-2018-15133)
COPY sites/default/settings.php sites/default/settings.php

# Step 3: Add a dummy Laravel .env file with plaintext APP_KEY and DB creds
COPY .env /var/www/html/.env

# Step 4: Set up MariaDB with weak root password and a user with reused credentials
COPY init-db.sql /docker-entrypoint-initdb.d/init-db.sql

# Step 5: Configure SSH with reused credentials (CVE-2016-6563)
RUN mkdir /var/run/sshd
COPY sshd_config /etc/ssh/sshd_config
COPY admin_authorized_keys /home/admin/.ssh/authorized_keys
RUN useradd -m admin && \
    echo 'admin:SuperSecretPass123!' | chpasswd && \
    mkdir -p /home/admin/.ssh && \
    chown -R admin:admin /home/admin/.ssh && \
    chmod 700 /home/admin/.ssh && \
    chmod 600 /home/admin/.ssh/authorized_keys

# Expose HTTP and SSH ports
EXPOSE 80 22

# Start Apache, MariaDB, and SSHD
CMD service mysql start && service ssh start && apache2-foreground
```

sites/default/settings.php
```php
<?php
// settings.php (Drupal config) - Vulnerable: Plaintext DB creds, world-readable
$databases['default']['default'] = array (
  'database' => 'drupal',
  'username' => 'admin',
  'password' => 'SuperSecretPass123!', // Vulnerability: Plaintext password, reused for SSH
  'host' => 'localhost',
  'driver' => 'mysql',
  'prefix' => '',
);

$config_directories = array();

$settings['hash_salt'] = 'dummyhashsalt';

// Dummy Laravel APP_KEY for CVE-2018-15133
$settings['laravel_app_key'] = 'base64:Q2hhbmdlTWVOb3c=';
```

.env
```env
# .env (Laravel-style) - Vulnerable: Exposed APP_KEY and DB creds
APP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:Q2hhbmdlTWVOb3c=
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=drupal
DB_USERNAME=admin
DB_PASSWORD=SuperSecretPass123! # Vulnerability: Same as SSH/admin user

# Dummy service account table reference
SERVICE_ACCOUNTS_TABLE=service_accounts
```

init-db.sql
```sql
-- init-db.sql
-- Vulnerable: Creates DB user with password reused for SSH, and a table documenting this
CREATE DATABASE IF NOT EXISTS drupal;
USE drupal;

CREATE USER 'admin'@'localhost' IDENTIFIED BY 'SuperSecretPass123!';
GRANT ALL PRIVILEGES ON drupal.* TO 'admin'@'localhost';

-- Table showing credential reuse (CVE-2016-6563)
CREATE TABLE service_accounts (
    id INT PRIMARY KEY AUTO_INCREMENT,
    service VARCHAR(50),
    username VARCHAR(50),
    password VARCHAR(100)
);

INSERT INTO service_accounts (service, username, password) VALUES
('mysql', 'admin', 'SuperSecretPass123!'),
('ssh', 'admin', 'SuperSecretPass123!');
```

sshd_config
```text
# sshd_config - Vulnerable: Allows password auth, weak admin password reused from DB
Port 22
PermitRootLogin no
PasswordAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
AllowUsers admin
```

admin_authorized_keys
```text
# admin_authorized_keys - Dummy public key for admin (optional, not required for password auth)
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC7dummykeyforadminuser
```

Directory structure:
```
Dockerfile
sites/default/settings.php
.env
init-db.sql
sshd_config
admin_authorized_keys
```

This setup exposes:
- Outdated Drupal (CVE-2019-6340) on port 80 for RCE.
- World-readable config files with plaintext, reused credentials (CVE-2018-15133).
- MariaDB with weak, reused credentials and a service_accounts table documenting the reuse (CVE-2016-6563).
- SSH server with password authentication enabled, using the same admin credentials as the database.

# End of answer