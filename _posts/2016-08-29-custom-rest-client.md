---
layout: post
title: "What to do with Java NoHttpResponseException"
tags: [devops, java]
---

The system I currently work on is built with [Spring Boot] and utilizes
[Eureka] as the service discovery mechanism. This has worked well for our team
except when we want to roll out changes without impacting users. We found that
one of the errors that would occur when deploying new code was the Java
`NoHttpResponseException`. We will explore briefly why this was happening
(based on our implementation) and how we are attempting to detect/prevent this
error.

So our implementation was as follows:

```java
package my.custom.package;

import org.apache.http.client.HttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestTemplate;

@Configuration
@PropertySource(value = "resttemplate.properties", ignoreResourceNotFound = true)
public class MyCustomRestTemplateConfiguration {
    @Value("${my.custom.rest.template.max.connections:500}")
    private Integer maxTotal;
    @Value("${my.custom.rest.template.default.max.per.host:50}")
    private Integer defaultMaxPerRoute;

    @Bean
    public RestTemplate restTemplate() {
        PoolingHttpClientConnectionManager manager = new PoolingHttpClientConnectionManager();
        manager.setDefaultMaxPerRoute(defaultMaxPerRoute);
        manager.setMaxTotal(maxTotal);
        HttpClient client = HttpClient.createMinimal(manager);
        return new RestTemplate(new HttpComponentsClientHttpRequestFactory(client));
    }
}
```

The piece that I would like to point out is the
`PoolingHttpClientConnectionManager`. This will open up and maintain a pool of
500 max HTTP connections with a maximum of 50 per host (IP address). With our
setup, we have three instances of each microservice application running and
registered with Eureka so each client will have some connections to each of the
three instances of that application.

This works great when IP addresses are not changing and the applications are
not being shutdown or spun up. When applications are being shutdown, they will
initiate a close on any connections that clients may have to them. This close
will result in a half open socket where the client (in Java) believes that the
socket is still open. However, at the C level, we can see that the socket is
actually in a `CLOSE_WAIT` socket status. This state of the socket cannot be
detected by Java and so Java assumes the socket is still viable. Upon a new
request being made, It will write to the socket and then read a `-1` from the
socket meaning that it failed to receive an HTTP Response from the server.

Now you may be wondering, "Why don't we just retry all
NoHttpResponseException?". This is a great question asked by many but
unfortunately, many will say to just retry the request. We decided for our
system that we should only retry requests that we know are safe. A correct
`NoHttpResponseException` would indicate that the server did in fact receive
the request but that it failed to respond to us (for any number of reasons). We
are left wondering now, "Is it safe to retry our request? What part of our
request did the server already process before encountering an error or did it
successful process the whole request but just fail to respond?" Because of
those scenarios, we wanted to sure that only errors where the socket was in the
`CLOSE_WAIT` state were retried because only these requests were guaranteed to
not have made it to the server. Below is the solution that we now use:

```java
package my.custom.package;

import org.apache.http.HttpClientConnection;
import org.apache.http.HttpException;
import org.apache.http.HttpRequest;
import org.apache.http.HttpResponse;
import org.apache.http.client.HttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.impl.conn.PoolingHttpClientConnectionManager;
import org.apache.http.protocol.HttpContext;
import org.apache.http.protocol.HttpRequestExecutor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestTemplate;

import java.io.IOException;
import java.net.SocketTimeoutException;

@Configuration
@PropertySource(value = "resttemplate.properties", ignoreResourceNotFound = true)
public class MyCustomRestTemplateConfiguration {
    @Value("${my.custom.rest.template.max.connections:500}")
    private Integer maxTotal;
    @Value("${my.custom.rest.template.default.max.per.host:50}")
    private Integer defaultMaxPerRoute;

    @Bean
    public RestTemplate restTemplate() {
        PoolingHttpClientConnectionManager manager = new PoolingHttpClientConnectionManager();
        manager.setDefaultMaxPerRoute(defaultMaxPerRoute);
        manager.setMaxTotal(maxTotal);
        HttpClient client = HttpClients.custom()
                .setConnectionManager(manager)
                .disableCookieManagement()
                .setRequestExecutor(new MyCustomHttpRequestExecutor())
                .build();
        return new RestTemplate(new HttpComponentsClientHttpRequestFactory(client));
    }

    private class MyCustomHttpRequestExecutor extends HttpRequestExecutor {
        private final HttpRequestExecutor requestExecutor = new HttpRequestExecutor();

        @Override
        public HttpResponse execute(
                final HttpRequest request,
                final HttpClientConnection conn,
                final HttpContext context) throws IOException, HttpException {
            int originalSocketTimeout = conn.getSocketTimeout();
            try {
                conn.setSocketTimeout(10);
                conn.receiveResponseHeader();
            } catch (SocketTimeoutException e) {
                // Socket is good to be used
            } catch (IOException e) {
                throw new IOException("Socket closed before writing!",
                        new MyCustomRetryableException("Socket closed before writing!"));
            } finally {
                conn.setSocketTimeout(originalSocketTimeout);
            }
            return requestExecutor.execute(request, conn, context);
        }
    }

    private class MyCustomRetryableException extends Throwable {
        MyCustomRetryableException(String s) {
            super(s);
        }
    }
}
```

We override the default `HttpRequestExecutor` that comes with the custom
HttpClient with our own custom executor. This does have performance
implications as every request will test the socket for 10 milliseconds now to
see if it is in the `CLOSE_WAIT` state.  If so, it will throw the `IOException`
and then throw our custom exception that we know is safe to retry. This test
allowed us to know that either the socket was good to use or we needed to
destroy that connection and use the next server in our list from Eureka. As
with all things in programming, there are tradeoffs and our desire to not have
errors returned to our users outweighed our needs for the performance hit that
we took with this implementation.

[Spring Boot]: http://projects.spring.io/spring-boot/
[Eureka]: https://github.com/Netflix/eureka
