1. Dockerfile (Dockerfile.struts)

```Dockerfile
FROM openjdk:8-jdk-alpine

RUN apk add --no-cache struts2

COPY struts-app /usr/local/struts-app

EXPOSE 80

CMD ["java", "-jar", "/usr/local/struts-app/struts2-json-plugin.jar", "--webapp", "/usr/local/struts-app"]
```

2. struts-app directory (struts-app)

```bash
struts-app/
├── WEB-INF/
│   ├── web.xml
│   └── lib/
│       └── struts2-json-plugin-2.3.31.jar
├── struts2-json-plugin.jar
```

3. web.xml (struts-app/WEB-INF/web.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         version="3.0">

    <action-mappings>
        <action name="/*" class="com.example.RceAction"/>
    </action-mappings>

</web-app>
```

4. RceAction.java (struts-app/src/main/java/com/example/RceAction.java)

```java
package com.example;

import com.opensymphony.xwork2.ActionSupport;
import java.io.PrintWriter;

public class RceAction extends ActionSupport {

    public String execute() throws Exception {
        return SUCCESS;
    }

    public void executeRce() throws Exception {
        PrintWriter writer = response.getWriter();
        writer.println("You've successfully exploited the Struts 2 RCE vulnerability!");
    }
}
```

5. Dockerfile (Dockerfile.ssh)

```Dockerfile
FROM alpine:latest

RUN apk add --no-cache openssh

COPY ssh-config /etc/ssh/sshd_config

EXPOSE 22

CMD ["sh", "-c", "echo 'root:password' >> /etc/passwd; sshd -D"]
```

6. ssh-config (ssh-config)

```bash
PermitRootLogin yes
```

7. Dockerfile (Dockerfile.mysql)

```Dockerfile
FROM mysql:5.7

ENV MYSQL_ROOT_PASSWORD password

EXPOSE 3306

CMD ["mysqld"]
```

These files and Dockerfiles will create three containers: a Struts container with the CVE-2017-5638 vulnerability, an SSH container with weak credentials, and a MySQL container with an unsecured root account. The attacker can exploit the Struts vulnerability to gain shell access, then use the weak SSH credentials to escalate privileges, and finally use the unsecured MySQL root account to access the database server.