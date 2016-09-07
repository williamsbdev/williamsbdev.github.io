---
layout: post
title: "Docker with Java and Nginx"
tags: [devops, docker, java, nginx]
---

The platform I work on is built with [Spring Boot] applications, hosted in a
[Docker] container and hidden behind an [Nginx] reverse proxy. All of these
items have been understood and tuned as we have worked and continue to work on
our platform to reduce downtime and responses resulting in an error (`5XX` HTTP
status codes).

We chose Docker as our method of deployment in order to take advantage of a
simplified host and isolation of each application for security and performance
reasons. Our Docker image is very minimal containing only a libc package needed
for the JVM and the Nginx static binary. Our entrypoint for the image is the
Java executable so our Java application is actually responsible for starting up
Nginx to be the reverse proxy for the Spring Boot application.

We wanted and needed Nginx because we were actually not able to see what the
embedded Tomcat server would respond with to the client connecting to the
service. We did have a request/response filter high in the chain but there were
still filters being applied by Tomcat that could/would change the response code
(example was a 501 when a garbage request method was sent). Now Nginx simply
proxies requests and does a log on the way out of the application after the
filter chain has been applied.

In order to have both processes running (Java Spring Boot App and Nginx), when
we are starting up the Java application in the container, it has knowledge of
where the Nginx binary is located in the Docker image so that it can start
Nginx once the application is ready to service requests. We achieved this by
implementing the `CommandLineRunner` interface.

```java
package my.custom.nginx;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.context.embedded.EmbeddedServletContainerCustomizer;
import org.springframework.boot.context.embedded.tomcat.EmbeddedServletContainerFactory;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.io.IOException;
import java.net.InetAddress;
import java.net.NetworkInterface;
import java.net.UnknownHostException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Enumeration;

@Configuration
public class JavaNginxConfig impements CommandLineRunner {
    private final Logger logger = LoggerFactory.getLogger(JavaNginxConfig.class);
    private Process start;

    @Bean
    public EmbeddedServletContainerCustomizer nginxCustomizer() {
        return container -> {
            try {
                container.setAddress(InetAddress.getByAddress(new byte[]{127, 0, 0, 1}));
            } catch (UnknownHostException e) {
                throw new RuntimeException("Failed to set address for TomcatContainer", e);
            }
            if (container instanceof TomcatEmbeddedServletContainerCustomizer) {
                final TomcatEmbeddedServletContainerFactory tomcat = (TomcatEmbeddedServletContainerFactory) container;
                tomcat.addConnectorCustomizers(connector -> connector.setPort(8181));
            }
        }
    }

    @Override
    public void run(String... args) throws Exception {
        try {
            configureListenDirectives();
        } catch (IOException e) {
            logger.error("Error occurred attempting to write the listen directives to the include file. Closing the context.");
            context.close();
        }

        try {
            start = new ProcessBuilder("nginx-binary-path/nginx", "-c", "nginx-conf-path/nginx.conf");
        } catch (IOException e) {
            logger.error("Error occurred attempting to start the Nginx process. Closing the context.");
            context.close();
        }
    }

    private void configureListenDirectives() throws IOException {
        Path includePath = Paths.get("nginx-conf/listen.include");
        Files.deleteIfExists(includePath);
        for (Enumeration<NetworkInterface> networkInterfaces = NetworkInterface.getNetworkInterfaces(); networkInterfaces.hasMoreElements();) {
            for (Enumeration<InetAddress> inetAddresses = networkInterfaces.nextElement().getInetAddresses(); inetAddresses.hasMoreElements();) {
                InetAddress inetAddress = inetAddresses.nextElement();
                if (!inetAddress.isLoopbackAddress()) {
                    Files.write(includePath,
                    ("listen " + inetAddress.toString().replaceAll("/", "") + ":8080;").getBytes());
                }
            }
        }
    }
}
```

What we found was that the `CommandLineRunner.run()` would not run until the
application was up and ready to service requests, Tomcat was running and
listening on our port inside the container on 8181. The other important piece
of this was that Tomcat was only listening inside the container. Listening on
`127.0.0.1` inside the container meant that we did not also bind to Docker IP
(typically `172.17.0.X`) so clients would have to use Nginx in order to
communicate with the Java application. Not having Nginx listening until Tomcat
was ready was also important so the container did not start accepting
connections, Nginx was started and listening but Tomcat was not, and fail with
a `502` (Bad Gateway).

All this work allowed us better logging of our true responses to clients of
each service, prevention of `502` errors that could result from prematurely
starting the Nginx process. We were previously annotating our `run` method with
`@PostConstruct` which would start Nginx right after the configuration was
created. This was very early in the application startup process and well before
Tomcat was ready and listening to service requests.

[Spring Boot]: http://projects.spring.io/spring-boot/
[Docker]: https://www.docker.com/
[Nginx]: https://www.nginx.com/
