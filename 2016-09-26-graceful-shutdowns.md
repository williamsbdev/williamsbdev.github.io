---
layout: post
title: "Graceful Shutdowns"
tags: [devops, java, aws]
---

The platform I currently work on is built with [Spring Boot] applications,
Mesos/Marathon/Zookeeper, RabbitMQ and hosted in [AWS]. In AWS we take
advantage of CloudFront, Elastic Load Balancers, and EC2. During some load
testing, we noticed that as we were deploying either new versions of the
applications or instances in our Mesos cluster, that we would get an assortment
of errors (`500` and `504` errors). We solved this by doing proper connection
draining and shutdown procedures at all levels of the stack.

Our Java applications are hosted in Docker containers that also have Nginx as a
reverse proxy. This was discussed in a [previous post]. The main problem with
our setup was that when the applications were being shutdown, not all the
outstanding requests in the application were finished before Spring Boot
started destroying beans. We solved this by implementing our own shutdown hooks
and chaining off of the work we did when we were starting up Nginx as the last
part of the application start procedure.

```java
package my.custom.nginx;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.context.embedded.EmbeddedServletContainerCustomizer;
import org.springframework.boot.context.embedded.tomcat.EmbeddedServletContainerFactory;
import org.springframework.context.ConfigurableApplicationContext;
import org.springframework.context.SmartLifecycle;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import my.custom.shutdown.MyCustomShutdownHook;

import java.io.IOException;
import java.net.InetAddress;
import java.net.NetworkInterface;
import java.net.UnknownHostException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.Enumeration;

@Configuration
public class JavaNginxConfig impements CommandLineRunner, SmartLifecycle, MyCustomShutdownHook {
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

    @Override
    public void runShutdownHook() {
        logger.info("Gracefully shutdown Nginx.");
        try {
            new ProcessBuilder("nginx-binary-path/nginx", "-c", "nginx-conf-path/nginx.conf", "-s", "quit").inheritIO().start();
        } catch (IOException e) {
            logger.error("Error occurred attempting to gracefully shutdown the Nginx process.");
        }
    }

    @Override
    public int getMyCustomShutdownHookOrder() {
        return 1;
    }

    @Override
    public boolean isAutoStartup() {
        return true;
    }

    @Override
    public void stop(Runnable callback) {
        logger.info("### Shutdown initiated ###");

        try {
            if(start != null) {
                start.waitFor();
                logger.info("### Nginx has shutdown all connections ###");
            }
            callback.run();
        } catch (final InterruptedException e) {
            logger.warn("InterruptedException", e);
        }
    }

    @Override
    public void start() {
        // unused method from Lifecycle interface
    }

    @Override
    public void stop() {
        // unused method (in favor of stop(Runnable callback)) from Lifecycle interface
    }

    @Override
    public boolean isRunning() {
        if(start != null) {
            return start.isAlive();
        }
        return true;
    }

    @Override
    public int getPhase() {
        return Integer.MAX_VALUE;
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

So in the above example, the `stop(Runnable callback)` is used for us to
trigger the shutdown of the Nginx process. This will tell Nginx to gracefully
shutdown and finish all of the outstanding requests it is currently handling.
In addition to that, because this implements the SmartLifecycle and sets the
`getPhase` to `Integer.MAX_VALUE`, it will be the last bean created and the
first one to be destroyed. This means that Nginx will not be started until the
application is started and tomcat is listening (preventing the `502` errors) and
since it is the first to be destroyed and waits for all Nginx worker processes
to finish before continuing with the shutdown sequence, Spring Boot will not
destroy beans in the middle of requests causing `500` errors.

Now that this is only part of the shutdown process. There is also the shutdown
process with the ELB that the application is registered with.

```java
package my.custom.aws.elb;

import com.amazonaws.regions.Regions;
import com.amazonaws.services.elasticloadbalancing.AmazonElasticLoadBalancingClient;
import com.amazonaws.services.elasticloadbalancing.model.DeregisterInstancesFromLoadBalancerRequest;
import com.amazonaws.services.elasticloadbalancing.model.DeregisterInstancesFromLoadBalancerResult;
import com.amazonaws.services.elasticloadbalancing.model.Instance;
import com.amazonaws.services.elasticloadbalancing.model.RegisterInstancesFromLoadBalancerRequest;
import com.amazonaws.services.elasticloadbalancing.model.RegisterInstancesFromLoadBalancerResult;
import com.amazonaws.util.EC2MetadataUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.PropertySource;
import my.custom.shutdown.MyCustomShutdownHook;

import java.util.Collections;

@Configuration
@PropertySource(value = "classpath:my-custom-elb.properties")
public class MyElbRegistrar implements CommandLineRunner, MyCustomShutdownHook {
    private static final Logger LOGGER = LoggerFactory.getLogger(MyElbRegistrar.class);

    @Value("${my.custom.elb-name}")
    private String elbName;

    private String instanceId;

    private AmazonElasticLoadBalancingClient client;

    private void init() {
        client = new AmazonElasticLoadBalancingClient().withRegion(Regions.getCurrentRegion());
        instanceId = EC2MetadataUtils.getInstanceId();
    }

    @Override
    public void run(String... args) throws Exception {
        LOGGER.info("Registering instance " + instanceId + " with ELB " + elbName);
        RegisterInstancesWithLoadBalancerRequest request = new RegisterInstancesWithLoadBalancerRequest(elbName, Collections.singletonList(new Instance(instanceId)));
        RegisterInstancesWithLoadBalancerResult result = client.registerInstancesWithLoadBalancer(request);
        LOGGER.info("Registration results: " + result);
    }

    @Override
    public int getMyCustomShutdownHookOrder() {
        return 0;
    }

    @Override
    public void runShutdownHook() {
        LOGGER.info("Deregistering instance " + instanceId + " with ELB " + elbName);
        DeregisterInstancesFromLoadBalancerRequest request = new DeregisterInstancesFromLoadBalancerRequest(elbName, Collections.singletonList(new Instance(instanceId)));
        DeregisterInstancesWithLoadBalancerResult result = client.deregisterInstancesFromLoadBalancer(request);
        LOGGER.info("Deregistration results: " + result);
    }

}
```

This is the class and the way the application registers itself with the ELB.
When the application is in a shutdown phase, it will first deregister with the
ELB and then proceed to gracefully shutdown Nginx (the
`getMyCustomShutdownHookOrder` from the `MyCustomShutdownHook` interface tells
the order in which to execute the `runShutdownHook`). The interface is provided below:

```java
package my.custom.shutdown;

public interface MyCustomShutdownHook {
    void runShutdownHook();
    int getMyCustomShutdownHookOrder();
}
```

This is used in the `MyCustomLifecycleService` which is provided below:

```java
package my.custom.lifecycle;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.event.AppliationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Service;
import my.custom.shutdown.MyCustomShutdownHook;

import java.util.Collections;
import java.util.Comparator;
import java.util.List;

@Service
public class MyCustomLifecycleService {
    private static final Logger LOGGER = LoggerFactory.getLogger(MyCustomLifecycleService.class);

    @Autowired
    private List<MyCustomShutdownHook> myCustomShutdownHooks;

    public MyCustomLifecycleService() {
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            LOGGER.warn("SHUTDOWN");
            for (MyCustomShutdownHook myCustomShutdownHook : getShutdownHooks()) {
                try {
                    LOGGER.warn("Running shutdown hook for '" + myCustomShutdownHook.getClass().getName() + "'.");
                    myCustomShutdownHook.runShutdownHook();
                    LOGGER.warn("Ran shutdown hook for '" + myCustomShutdownHook.getClass().getName() + "'.");
                } catch (Exception e) {
                    LOGGER.error("Exception thrown running shutdown hook for '" + myCustomShutdownHook.getClass().getName() + "'.");
                }
            }
        }));
    }

    @EventListener
    public void onStartup(ApplicationReadyEvent event) {
        Logger.warn("STARTUP");
    }

    private List<MyCustomShutdownHook> getShutdownHooks() {
        return Collections.sort(myCustomShutdownHooks, Comparator.comparingInt(MyCustomShutdownHook::getMyCustomShutdownHookOrder));
    }
}

```

This final class is the key to having all the shutdown processes run in the
correct order. However, all pieces were needed for the Java applications to
shutdown in a graceful way (finish the outstanding requests before destroying
beans/terminating connections). In addition to all this work, the ELBs are also
configured for 60 second connection draining to ensure that requests can
successfully be returned before the ELB will shutdown it's connections to the
applications, otherwise we would continue to see the Nginx errors of `499`.

[Spring Boot]: http://projects.spring.io/spring-boot/
[AWS]: https://aws.amazon.com/
[previous post]: https://williamsbdev.com/posts/docker-with-java-and-nginx
