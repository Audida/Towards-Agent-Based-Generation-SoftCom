Dockerfile
```
# Vulnerable Environment: Drupal (CVE-2019-6340) + Laravel-style .env leak (CVE-2018-15133) + MariaDB (CVE-2016-6663)
FROM php:7.2-apache

# --- Step 1: Outdated Drupal with RCE (CVE-2019-6340) ---
# Install dependencies
RUN apt-get update && \
    apt-get install -y wget unzip mariadb-server && \
    rm -rf /var/lib/apt/lists/*

# Download vulnerable Drupal 8.6.9 (vulnerable to CVE-2019-6340)
WORKDIR /var/www/html
RUN wget -q https://ftp.drupal.org/files/projects/drupal-8.6.9.zip && \
    unzip drupal-8.6.9.zip && \
    mv drupal-8.6.9/* . && \
    mv drupal-8.6.9/.htaccess . && \
    rm -rf drupal-8.6.9 drupal-8.6.9.zip

# Set permissions (world-writable for demonstration, insecure)
RUN chmod -R 777 /var/www/html

# Enable Apache mod_rewrite for Drupal
RUN a2enmod rewrite

# --- Step 2: World-readable .env file with dummy credentials (CVE-2018-15133) ---
# Place a Laravel-style .env file in the web root (insecure, world-readable)
COPY .env /var/www/html/.env
RUN chmod 644 /var/www/html/.env

# --- Step 3: Vulnerable MariaDB (CVE-2016-6663) ---
# Initialize MariaDB with insecure settings and a vulnerable version
# (MariaDB 10.1.21 is vulnerable to CVE-2016-6663)
RUN apt-get update && \
    apt-get install -y wget lsb-release && \
    echo "deb [arch=amd64] http://archive.mariadb.org/mariadb-10.1.21/repo/debian $(lsb_release -cs) main" > /etc/apt/sources.list.d/mariadb.list && \
    apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y mariadb-server=10.1.21+maria-1~$(lsb_release -cs) && \
    rm -rf /var/lib/apt/lists/*

# Copy vulnerable MariaDB config (bind to localhost, weak permissions)
COPY my.cnf /etc/mysql/my.cnf

# Add dummy database and user for demonstration
COPY db-init.sql /docker-entrypoint-initdb.d/db-init.sql

# Expose web (80) and database (3306) ports
EXPOSE 80 3306

# Start both Apache and MariaDB (insecure, for demo only)
COPY start.sh /start.sh
RUN chmod +x /start.sh

CMD ["/start.sh"]
```

.env
```
# --- Step 2: World-readable .env file (CVE-2018-15133) ---
APP_NAME=VulnerableApp
APP_ENV=local
APP_KEY=base64:dummykey=
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=vulndb
DB_USERNAME=vulnuser
DB_PASSWORD=SuperSecretPassword123
```

my.cnf
```
# --- Step 3: MariaDB config with weak permissions (CVE-2016-6663) ---
[mysqld]
user            = mysql
bind-address    = 127.0.0.1
datadir         = /var/lib/mysql
socket          = /var/run/mysqld/mysqld.sock
# Insecure: Allow LOAD DATA LOCAL INFILE (can be abused)
local-infile    = 1
# Insecure: Skip symbolic-links protection
symbolic-links=0

[client]
port            = 3306
socket          = /var/run/mysqld/mysqld.sock
```

db-init.sql
```
-- --- Step 3: Dummy database and user for MariaDB ---
CREATE DATABASE IF NOT EXISTS vulndb;
CREATE USER 'vulnuser'@'localhost' IDENTIFIED BY 'SuperSecretPassword123';
GRANT ALL PRIVILEGES ON vulndb.* TO 'vulnuser'@'localhost';
FLUSH PRIVILEGES;

USE vulndb;
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50),
    password VARCHAR(255),
    email VARCHAR(100)
);

INSERT INTO users (username, password, email) VALUES
('admin', 'adminpass', 'admin@example.com'),
('user', 'userpass', 'user@example.com');
```

start.sh
```bash
#!/bin/bash
# --- Start MariaDB and Apache in foreground (insecure, for demo only) ---
service mysql start
apache2-foreground
```

# File List:
- Dockerfile
- .env
- my.cnf
- db-init.sql
- start.sh

# Notes:
- Drupal 8.6.9 is used for CVE-2019-6340 (RCE).
- The .env file is world-readable and contains dummy DB credentials (CVE-2018-15133).
- MariaDB 10.1.21 is vulnerable to CVE-2016-6663 (privilege escalation).
- All files include comments indicating which vulnerability they simulate.
- This setup is intentionally insecure and for educational/lab use only.