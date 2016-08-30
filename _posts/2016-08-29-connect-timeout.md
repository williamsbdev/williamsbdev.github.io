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
        HttpComponentsClientHttpRequestFactory requestFactory = new HttpComponentsClientHttpRequestFactory(client)
        requestFactory.setConnectTimeout(5000);
        return new RestTemplate(requestFactory);
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
