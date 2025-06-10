Dockerfile
```Dockerfile
# Dockerfile: Intentionally vulnerable environment simulating the described attack chain

FROM php:7.2-apache

# --- Step 1: Expose HTTP port and install vulnerable WordPress (CVE-2019-8942) ---
# WordPress <5.0.1 is vulnerable to RCE via file upload
RUN apt-get update && \
    apt-get install -y wget unzip mariadb-client libpng-dev libjpeg-dev libfreetype6-dev libzip-dev git openssh-server && \
    docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ && \
    docker-php-ext-install gd zip mysqli

# Download and extract vulnerable WordPress version
WORKDIR /var/www/html
RUN wget https://wordpress.org/wordpress-5.0.zip && \
    unzip wordpress-5.0.zip && \
    cp -r wordpress/* . && \
    rm -rf wordpress wordpress-5.0.zip

# --- Step 2: Add Laravel app with hard-coded credentials (CVE-2018-15133) ---
# Laravel <5.7.7 with default APP_KEY in .env
RUN cd /var/www && \
    git clone --branch v5.7.6 https://github.com/laravel/laravel.git laravel-app && \
    cd laravel-app && \
    composer install --no-interaction --no-scripts && \
    chown -R www-data:www-data /var/www/laravel-app

# Copy vulnerable .env file with default APP_KEY and hard-coded DB credentials
COPY laravel.env /var/www/laravel-app/.env

# --- Step 3: Install vulnerable libssh (CVE-2018-10933) and add dummy SSH keys ---
RUN apt-get update && \
    apt-get install -y build-essential cmake pkg-config && \
    wget https://www.libssh.org/files/0.7/libssh-0.7.6.tar.xz && \
    tar xf libssh-0.7.6.tar.xz && \
    cd libssh-0.7.6 && \
    cmake . && make && make install && \
    ldconfig && \
    cd .. && rm -rf libssh-0.7.6 libssh-0.7.6.tar.xz

# Enable and configure SSH server with weak settings
RUN mkdir /var/run/sshd && \
    echo 'root:toor' | chpasswd && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/UsePAM yes/UsePAM no/' /etc/ssh/sshd_config

# Add dummy SSH keys for lateral movement simulation
COPY id_rsa /root/.ssh/id_rsa
COPY id_rsa.pub /root/.ssh/id_rsa.pub
RUN chmod 600 /root/.ssh/id_rsa && chmod 644 /root/.ssh/id_rsa.pub && \
    mkdir -p /home/www-data/.ssh && \
    cp /root/.ssh/id_rsa /home/www-data/.ssh/id_rsa && \
    cp /root/.ssh/id_rsa.pub /home/www-data/.ssh/id_rsa.pub && \
    chmod 600 /home/www-data/.ssh/id_rsa && chmod 644 /home/www-data/.ssh/id_rsa.pub && \
    chown -R www-data:www-data /home/www-data/.ssh

# --- Expose ports ---
EXPOSE 80 22

# --- Start Apache and SSHD ---
CMD service ssh start && apache2-foreground
```

laravel.env
```env
# .env file for Laravel (CVE-2018-15133)
# Vulnerability: Default APP_KEY and hard-coded DB credentials

APP_NAME=Laravel
APP_ENV=local
APP_KEY=base64:SomeRandomStringSomeRandomStringSomeRandomString==
APP_DEBUG=true
APP_URL=http://localhost

LOG_CHANNEL=stack

DB_CONNECTION=mysql
DB_HOST=localhost
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=supersecretpassword  # <-- Hard-coded credential

BROADCAST_DRIVER=log
CACHE_DRIVER=file
QUEUE_DRIVER=sync
SESSION_DRIVER=file
SESSION_LIFETIME=120
```

id_rsa
```plaintext
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAzv0Q4w8h1k0n6Rz2F1v5vQZxQn8F8Qk9d4n8yT8l6Q8n8Qk9
d4n8yT8l6Q8n8Qk9d4n8yT8l6Q8n8Qk9d4n8yT8l6Q8n8Qk9d4n8yT8l6Q8n8Qk9
d4n8yT8l6Q8n8Qk9d4n8yT8l6Q8n8Qk9d4n8yT8l6Q8n8Qk9d4n8yT8l6Q8n8Qk9
d4n8yT8l6Q8n8Qk9d4n8yT8l6Q8n8Qk9d4n8yT8l6Q8n8Qk9d4n8yT8l6Q8n8Qk9
d4n8yT8l6Q8n8Qk9d4n8yT8l6Q8n8Qk9d4n8yT8l6Q8n8Qk9d4n8yT8l6Q8n8Qk9
-----END RSA PRIVATE KEY-----
```

id_rsa.pub
```plaintext
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDO/RDjDyHWTSerHPYXW/m9BnFCfwXxCT13ifzJPyXpD
yf
xCT13ifzJPyXpDyfxCT13ifzJPyXpDyfxCT13ifzJPyXpDyfxCT13ifzJPyXpDyfxCT13ifzJPyXpDyf
xCT13ifzJPyXpDyfxCT13ifzJPyXpDyfxCT13ifzJPyXpDyfxCT13ifzJPyXpDyfxCT13ifzJPyXpDyf
xCT13ifzJPyXpDyfxCT13ifzJPyXpDyfxCT13ifzJPyXpDyfxCT13ifzJPyXpDyfxCT13ifzJPyXpDyf
xCT13ifzJPyXpDyfxCT13ifzJPyXpDyfxCT13ifzJPyXpDyfxCT13ifzJPyXpDyf test@vulnerable
```

# File List

- Dockerfile (above)
- laravel.env (above)
- id_rsa (dummy private key, above)
- id_rsa.pub (dummy public key, above)

# Notes

- The Dockerfile sets up a vulnerable WordPress (CVE-2019-8942) and a Laravel app with a default APP_KEY and hard-coded DB credentials (CVE-2018-15133).
- It installs a vulnerable version of libssh (CVE-2018-10933) and configures SSH with weak settings and dummy keys for lateral movement simulation.
- All files are intentionally misconfigured or contain dummy secrets to simulate real-world exploitation chains.
- The SSH keys are dummy and for demonstration only; replace with real test keys if needed for actual lab use.
- The container exposes both HTTP (80) and SSH (22) for attack simulation.

This setup enables a full attack chain as described: initial RCE via WordPress, privilege escalation via Laravel config, and lateral movement/data exfiltration via SSH.