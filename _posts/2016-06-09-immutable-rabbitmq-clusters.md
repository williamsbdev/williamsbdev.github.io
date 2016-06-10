---
layout: post
title: "Immutable RabbitMQ Clusters"
tags: [aws, devops, rabbitmq]
---

### Forward

[RabbitMQ] clustering can be easy to setup. Setting up a cluster via [immutable]
servers, can be tricky. Using AWS with an auto scaling group and having the
nodes join/leave the cluster is problematic. The goal is allow RabbitMQ to be
just like the rest of your infrastructure that is immutable.

### Assumptions

- Our platform is 100% AWS - automated all through cloudformation
- We have a three node RabbitMQ cluster in a single region (us-west-2) spread
  across three Availability Zones (AZs) with a subnet assigned to each AZ
- Two [Auto Scaling Groups] with two [Launch Configurations]
  - A primary auto scaling group and launch configuration with a single node in
    a single AZ/subnet
  - A secondary auto scaling group and launch configuration with two nodes
    spread across two AZs/subnets that is dependent on the primary auto scaling
    group
- Predefined hostnames (we will assume `rabbitmq0`, `rabbitmq1`, and
  `rabbitmq2`) based on the subnet where each node resides

### Checks to ensure RabbitMQ node is healthy and operational

The nodes will only send a [cfn-signal] once they have either started a
cluster, or successfully joined the cluster which is defined by joining a
cluster where all the `ha-mode: all`, meaning that all nodes have all queues,
and the `ha-sync-mode: automatic`, meaning that new nodes have the queues
synchronized them ([RabbitMQ high availability configurations]).

### Creating an environment

The first auto scaling group contains the primary RabbitMQ node that will start
the cluster if it cannot discover the other two. All of this is done via some
upstart scripts that wrap RabbitMQ and provide the logic of whether to start a
new cluster or join an existing one. All RabbitMQ nodes will check the
predefined hostnames that are already configured via the
`/etc/rabbitmq/rabbitmq.config` in the `cluster_nodes` section. The first node,
`rabbitmq0`, will not see any other nodes running yet and will start the
cluster. Checking for the cluster is done via the following command for each
node in the cluster (`rabbitmq0-2`):

    rabbitmqctl -n rabbitmq0 status 2>&1 | grep 'Error'

If this command above returns an empty string, then we know that RabbitMQ is
running on that node and should be part of a cluster. If this command is able
to find the `Error` text we expected, we assume that the node is unreachable
and start the cluster.

On a create, once the first node has found no other running RabbitMQ and has
started the cluster, it will send the cfn-signal (because it has successfully
started the cluster and there are no queues defined yet to synchronize) so that
the second auto scaling group can create the other two secondary nodes in the
cluster. Both of these nodes will start up with the same predefined
configuration and find that the first node has already started the cluster. The
nodes will then join that cluster and cfn-signal, again because there are no
queues defined, the synchronization is "free".

### Updating an environment

When applying an update via AWS CloudFormation, it will roll through the auto
scaling groups/launch configurations one instance at a time. This is achieved
through the following configuration.

The primary group has:

  1. MaxSize of 1 (number of instances to be running at any one time)
  2. MinSize of 0 instances
  3. DesiredCapacity of 1
  4. MaxBatchSize of 1 (defined via the UpdatePolicy)
  5. MinInstancesInService of 0 (defined via the UpdatePolicy)
  6. WaitOnResourceSignals of true (defined via the UpdatePolicy)

The secondary group has:

  1. MaxSize of 2
  2. MinSize of 1 instances
  3. DesiredCapacity of 2
  4. MaxBatchSize of 1 (defined via the UpdatePolicy)
  5. MinInstancesInService of 1 (defined via the UpdatePolicy)
  6. WaitOnResourceSignals of true (defined via the UpdatePolicy)

The most important pieces of the configurations above are a combination of the
MaxSize of the auto scaling group and the MinInstancesInService of the
UpdatePolicy. These two together, tell CloudFormation to do an N-1 rolling
model instead of the normal N+1 model that is default. So, as we destroy a node
of the cluster, the hostname is now "available" for us to reuse on the new
instance that will be created in that subnet.

As the auto scaling groups roll through the instances, each instance will see
that there is already an existing cluster, join that cluster, synchronize the
queues and send the cfn-signal once all that is done.

### Final Thoughts

At the end of the day, we found a way that even RabbitMQ, which seems to be
treated in the community as a pet server that should be managed manually, can
be automated to come and go as we roll out changes. We can treat these servers
as immutable and once deployed, never touch the server. If the hardware is bad
on one of the machines (which can happen in AWS), simply destroying the
instance and letting the auto scaling group create a new instance to join the
cluster is perfectly normal. The predictable/repeatable nature of the RabbitMQ
nodes being created or updated makes the extra work done upfront worthwhile.


[RabbitMQ]: https://www.rabbitmq.com
[immutable]: https://www.google.com/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=define%20immutable
[Auto Scaling Groups]: http://docs.aws.amazon.com/autoscaling/latest/userguide/WhatIsAutoScaling.html
[Launch Configurations]: http://docs.aws.amazon.com/autoscaling/latest/userguide/LaunchConfiguration.html
[cfn-signal]: http://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-signal.html
[RabbitMQ high availability configurations]: https://www.rabbitmq.com/ha.html
