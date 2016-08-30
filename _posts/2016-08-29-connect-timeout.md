---
layout: post
title: "Connect Timeout defaults"
tags: [devops, java]
---

In my [previous post], I showed how we implemented our own custom RestTemplate
for all rest requests. There was a default setting I found that caused
connections to timeout and thus have the load balancers return a `504
GATEWAY_TIMEOUT`. It was

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
        HttpComponentsClientHttpRequestFactory requestFactory = new HttpComponentsClientHttpRequestFactory(client);
        return new RestTemplate(requestFactory);
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

The setting that I added was to the `requestFactory`. By default, it would wait
indefinitely to connect which would cause some of my connections to never
timeout thus fail back out of API as a `504` HTTP status code to consumers. In
tandem with this setting change, we also implemented retry logic to the next
server when we received a `ConnectTimeoutException` (which will be covered in
another post).

[previous post]: https://williamsbdev.com/posts/no-http-response-exceptions
