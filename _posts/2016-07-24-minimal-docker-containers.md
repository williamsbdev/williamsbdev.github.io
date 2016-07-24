---
layout: post
title: "Minimal Docker Images"
tags: [aws, devops, docker]
---

[Docker] is a great tool. One of my favorite aspects of building,
shipping, and running docker containers is the lightweight nature of
deployment. Only that which is needed for the container should be included in
the image. The problem is that many running containers include more than what
is absolutely necesary for the application to run.

Many of the base images that I see people using for their deployable images
have a full OS such as Ubuntu, Debian, or Alpine (although Alpine does have a
much smaller footprint). Docker image layering is great and strongly
encouraged. I can create my own MySQL image based on the community supported
MySQL image from [DockerHub]. I can then add any other customization I need in
my `Dockerfile` (see below):

```bash
FROM mysql:5.6

RUN ./do-custom-stuff.sh
```

I recently pulled the `node:4` image from DockerHub. It was `656.9 MB` and I had
not even added my application! If we trace the history of the this image, we
will find that it is based on the `debian:jessie` image. To run a node
application, I do not need a full OS.

At work, I work with some really smart people. We write and deploy many Java
[Spring Boot] applications. We compile them into a [fat jar] and then we have a
docker image that only has the [libc] package needed for the JVM and then on
top of that, we trimmed out the portions of the JVM that we do not need. The
base image for our deployable images is around 100 MB. We will add roughly
another 60 MB to the image with the Spring Boot jar that includes all our
dependencies. So our deployable images are right around 160 MB. This still
seems large when I compare to other images and see that a statically compiled
[Go] application can be right around 20 MB.

These minimal images are nice for deploying to production as they do not have
any unnecesary items in the image and will result in faster deployments as it
is a smaller image to pull from the registry. With all things though, there are
pros and cons each way. The minimal images are not nice for local development
if you need to do any sort of debugging. Having a shell in the container to
investigate is super handy but is most likely unnecessary in production. As for
me, I am always to going to strive to keep my deployable images to a minimum
and include only what is absolutely necessary to run my application.

[Docker]: https://www.docker.com/
[DockerHub]: https://hub.docker.com/
[Spring Boot]: http://projects.spring.io/spring-boot/
[fat jar]: http://stackoverflow.com/questions/19150811/what-is-a-fat-jar
[libc]: http://packages.ubuntu.com/precise/libc6
[Go]: https://golang.org/
