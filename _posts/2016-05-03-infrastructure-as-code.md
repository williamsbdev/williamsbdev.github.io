---
layout: post
title: "Infrastructure as code"
tags: [aws, devops, docker, mesos, marathon, zookeeper, rabbitmq]
---

Infrastructure as code. This buzz phrase is being thrown around and referenced
by many reputable entities/persons.  [ThoughtWorks], [Devops.com], [Puppet],
[Martin Fowler], and [O'Reilly Publishing] just to name a few of the top Google
hits. This phrase is not just words that we say on our team. We believe this is
important. To us, infrastructure as code means that any change, software,
styling, firewall, system package, upstart script, etc., starts at the
developer laptop on a Git branch. The change is made in source control, tested,
and submitted as a pull request (PR) for peer review.

High level overview of our stack includes:

- 100% [AWS] with near 100% of AWS resources defined through CloudFormation
- [AWS RDS with MySQL] for database storage
- [RabbitMQ] used for messaging
- [Mesos] cluster supported by [Zookeeper] and [Marathon]
- [Docker] containers that contain our Java [Spring Boot] jars
- [Ember.js] for the UI/UX of our customer facing applications

One more thing to clarify as far as terminology for our team. Our definition of
environment is an AWS [VPC] which contains everything needed to be a production
class system. This means that it will contain an RDS instance/cluster, a Mesos
Cluster, a RabbitMQ cluser, and be running the necessary software to service
requests to our platform.

When we want to build a new environment or deploy a change, we check out the
desired Git commit SHA from our single source code repository and build an
environment with [Jenkins] doing the heavy lifting. We have created several
Jenkins jobs that feed in the appropriate parameters to the scripts checked
into the source code.  These will create the CloudFormation for our
infrastructure, deploy the queue definitions, deploy the docker container
versions/configuration via Marathon, and check the health of everything once
completed.

Our process described above removes manual interactions. While we do have
access to many interfaces that would allow us to make changes quickly, we as a
team value the predictable/repeatable nature of our system. Before a change is
deployed, our build pipe will build that environment run our full suite of unit
and black box API functional tests against the environment to ensure all
functionality works as expected. Tracking every change to our system via a Git
commit, allows us to go back and know exactly when a change was made and (with
good commit messages) for what reason the change was made. The PR process also
ensures accountability and ownership of the team. In the end, everyone on the
team is empowered to make a change to the system, the security team likes being
able to see every change that was made to the system, and the team as a whole
has confidence in building/maintaining our platform/system.

[ThoughtWorks]: https://www.thoughtworks.com/insights/blog/infrastructure-code-reason-smile
[Devops.com]: http://devops.com/2014/05/05/meet-infrastructure-code/
[Puppet]: https://puppet.com/solutions/infrastructure-as-code
[Martin Fowler]: http://martinfowler.com/bliki/InfrastructureAsCode.html
[O'Reilly Publishing]: http://infrastructure-as-code.com/
[AWS]: https://aws.amazon.com/
[AWS RDS with MySQL]: https://aws.amazon.com/rds/mysql/
[RabbitMQ]: https://www.rabbitmq.com/
[Mesos]: https://mesosphere.com/
[Zookeeper]: https://zookeeper.apache.org/
[Marathon]: https://mesosphere.github.io/marathon/
[Docker]: https://www.docker.io
[Ember.js]: http://emberjs.com
[VPC]: https://aws.amazon.com/vpc/
[Jenkins]: https://jenkins.io/
