Dockerfile:
```Dockerfile
# Base image with Apache Struts, Elasticsearch, and OpenSSH
FROM alpine:latest

# Install Apache Struts, Elasticsearch, and OpenSSH
RUN apk add --no-cache struts2 elasticsearch openjdk8 openjdk8-jre openssh

# Expose ports for HTTP, SSH, and Elasticsearch
EXPOSE 80 22 9200

# Set working directory
WORKDIR /app

# Copy Struts web application with CVE-2017-5638
COPY struts-app /app

# Copy Elasticsearch configuration with CVE-2019-7601
COPY elasticsearch.yml /etc/elasticsearch/

# Copy SSH default configuration with weak credentials
COPY ssh /etc/ssh/

# Run Elasticsearch and Apache Struts on startup
CMD ["/etc/init.d/elasticsearch start", "java -jar /app/struts2-core-2.5.14.jar"]

# Struts web application with CVE-2017-5638
cat << EOF > /app/struts-app/WEB-INF/struts.xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE struts PUBLIC
    "-//Apache Software Foundation//DTD Struts Configuration 2.0//EN"
    "http://struts.apache.org/dtds/struts-2.0.dtd">
<struts>
    <constant name="struts.devMode" value="true"/>
    <package name="default" namespace="/" extends="struts-default">
        <action name="*">
            <result>/index.jsp</result>
        </action>
    </package>
</struts>
EOF

# Elasticsearch configuration with CVE-2019-7601
cat << EOF > /etc/elasticsearch/elasticsearch.yml
# ... (other configuration options)
http.host: 0.0.0.0
http.port: 9200
xpack.security.enabled: false
EOF

# SSH default configuration with weak credentials
cat << EOF > /etc/ssh/sshd_config
# ... (other configuration options)
PermitRootLogin yes
# Set a weak password for root user
PasswordAuthentication yes
RootPassword root
EOF
```

Additional files:

1. struts-app (directory): Contains the Struts web application with CVE-2017-5638.

2. elasticsearch.yml: Elasticsearch configuration file with CVE-2019-7601.

3. ssh (directory): Contains the SSH default configuration file with weak credentials.