1. Dockerfile for Apache Struts with CVE-2017-5638:

```Dockerfile
FROM openjdk:8-jdk-alpine

RUN apk add --no-cache --virtual .build-deps gcc openjdk8-dev maven

WORKDIR /app

COPY pom.xml /app
COPY struts2-example /app

RUN mvn clean install

EXPOSE 8080

CMD ["java", "-jar", "struts2-example/target/struts2-example-1.0-SNAPSHOT.jar"]
```

2. struts2-example/pom.xml (Maven project configuration):

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>struts2-example</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.apache.struts</groupId>
            <artifactId>struts2-core</artifactId>
            <version>2.3.31.1</version>
        </dependency>
    </dependencies>
</project>
```

3. struts2-example/src/main/java/com/example/ActionBean.java (Vulnerable Struts action class):

```java
package com.example;

import com.opensymphony.xwork2.ActionSupport;
import org.apache.struts2.ServletActionContext;

public class ActionBean extends ActionSupport {
    private String param;

    public String execute() throws Exception {
        ServletActionContext.getResponse().getWriter().print("You entered: " + param);
        return SUCCESS;
    }

    public String getParam() {
        return param;
    }

    public void setParam(String param) {
        this.param = param;
    }
}
```

4. struts2-example/src/main/webapp/WEB-INF/struts.xml (Struts configuration file):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE struts PUBLIC
 "-//Apache Software Foundation//DTD Struts Configuration 2.0//EN"
 "http://struts.apache.org/dtds/struts-2.0.dtd">

<struts>
    <action name="actionBean" class="com.example.ActionBean">
        <result name="success">/success.jsp</result>
    </action>
</struts>
```

5. struts2-example/src/main/webapp/WEB-INF/web.xml (Web application configuration file):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://java.sun.com/xml/ns/javaee"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
         id="WebApp_ID" version="2.5">

    <display-name>Struts2 Example</display-name>

    <filter>
        <filter-name>struts2</filter-name>
        <filter-class>org.apache.struts2.dispatcher.ng.filter.StrutsPrepareAndExecuteFilter</filter-class>
    </filter>

    <filter-mapping>
        <filter-name>struts2</filter-name>
        <url-pattern>/*</url-pattern>
    </filter-mapping>
</web-app>
```

6. Dockerfile for MongoDB with no authentication:

```Dockerfile
FROM mongo:latest

RUN echo "mongodb-org-service-runner" | sudo tee -a /etc/group && \
    echo "mongodb-org-service" | sudo tee -a /etc/group && \
    echo "mongodb-org-service ALL=(ALL:ALL) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/mongodb-org-service

RUN chmod 04755 /usr/bin/mongod

CMD ["mongod", "--auth", "--bind_ip", "0.0.0.0"]
```

7. Dockerfile for SSH server with default credentials:

```Dockerfile
FROM alpine:latest

RUN apk add --no-cache openSSH

COPY ssh-config /etc/ssh/sshd_config

CMD ["sshd", "-D"]

# ssh-config file content:
PermitRootLogin yes
PasswordAuthentication yes
```

8. Optional: A script to automate the setup and start of the containers.

```bash
#!/bin/bash

# Build and run the Apache Struts container
docker build -t struts-vulnerable . && docker run -d -p 8080:8080 struts-vulnerable

# Build and run the MongoDB container
docker build -t mongodb-vulnerable . && docker run -d -p 27017:27017 mongodb-vulnerable

# Build and run the SSH container
docker build -t ssh-vulnerable . && docker run -d -p 22:22 ssh-vulnerable
```