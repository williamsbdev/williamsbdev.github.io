---
layout: post
title: "Hotplug AWS Network Interfaces"
tags: [aws, devops]
---

A [friend] and I were working together and discovered we had a need to be able
to attach our AWS network interfaces from one instance to another. So like any
good OSS developer, we looked for a project that already did this for us. What
we found was this [project]. We will start by walking through the project to
see how it works.

The first thing we start with is the [53-ec2-network-interfaces.rules] files which are located in
`/etc/udev/rules.d/` (contents below).

```bash
# /etc/udev/rules.d/53-ec2-network-interfaces.rules

# Copyright (C) 2012 Amazon.com, Inc. or its affiliates.
# All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License").
# You may not use this file except in compliance with the License.
# A copy of the License is located at
#
#    http://aws.amazon.com/apache2.0/
#
# or in the "license" file accompanying this file. This file is
# distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS
# OF ANY KIND, either express or implied. See the License for the
# specific language governing permissions and limitations under the
# License.

ACTION=="add", SUBSYSTEM=="net", KERNEL=="eth*", IMPORT{program}="/bin/sleep 1"
SUBSYSTEM=="net", RUN+="/etc/network/ec2net.hotplug"
```

As we are looking at what this file does, it is saying that every time there is
an `add` action from the `net` Linux Network subsystem for the `eth` kernel
module, it is going to sleep for 1 second and then add the
`/etc/network/ec2net.hotplug` script to the run list for the `net` subsystem.
So now let's look at the [ec2net.hotplug] script to see what is going to run.

```bash
# /etc/network/ec2net.hotplug

#!/bin/bash

# Copyright (C) 2012 Amazon.com, Inc. or its affiliates.
# All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License").
# You may not use this file except in compliance with the License.
# A copy of the License is located at
#
#    http://aws.amazon.com/apache2.0/
#
# or in the "license" file accompanying this file. This file is
# distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS
# OF ANY KIND, either express or implied. See the License for the
# specific language governing permissions and limitations under the
# License.

# During init and before the network service is started, metadata is not
# available. Exit without attempting to configure the elastic interface
# if eth0 isn't up yet.
if [ ! -f /sys/class/net/eth0/operstate ] || ! grep 'up' /sys/class/net/eth0/operstate; then
  exit
fi

. /etc/network/ec2net-functions

case $ACTION in
  add)
    plug_interface
    ;;
  remove)
    unplug_interface
    ;;
esac
```

So in this script, it is going to ensure that the network interface `eth0` is
available before it does a switch on the actions `add` or `remove` from the
`net` subsystem. It will then invoke the `plug_interface` or `unplug_interface`
function which comes from the [ec2net-functions] script. Let's take a look at
the `plug_interface` function to understand how we are going to configure the
interface.

```bash
# excerpt from /etc/network/ec2net-functions

plug_interface() {
    logger "ABOUT TO CALL REWRITE PRIMARY"
  rewrite_primary
}
```

This is not exciting as it tells us we're just going to call the
`rewrite_primary` function now.

```bash
# excerpt from /etc/network/ec2net-functions

rewrite_primary() {
  if [ "${INTERFACE}" == "eth0" ]; then
    return
  fi
  cidr=$(get_cidr)
  if [ -z ${cidr} ]; then
    return
  fi
  network=$(echo ${cidr}|cut -d/ -f1)
  router=$(( $(echo ${network}|cut -d. -f4) + 1))
  gateway="$(echo ${network}|cut -d. -f1-3).${router}"
  cat <<- EOF > ${config_file}
# This file is automaticatically generated
# See https://github.com/jessecollier/ubuntu-ec2net for source
auto ${INTERFACE}
iface ${INTERFACE} inet dhcp
post-up ip route add default via ${gateway} dev ${INTERFACE} table ${RTABLE}
post-up ip route add default via ${gateway} dev ${INTERFACE} metric ${RTABLE}
EOF
  # Use broadcast address instead of unicast dhcp server address.
  # Works around an issue with two interfaces on the same subnet.
  # Unicast lease requests go out the first available interface,
  # and dhclient ignores the response. Broadcast requests go out
  # the expected interface, and dhclient accepts the lease offer.
  cat <<- EOF > ${dhclient_file}
  supersede dhcp-server-identifier 255.255.255.255;
EOF
}
```

In this function we can see that we are going to skipping anything for the
`eth0` (default) interface. We will then guarantee that we can get a CIDR (the
ipv4 CIDR block for the EC2 instance) before doing any other processing. From
that CIDR we will be able to determine the gateway for the new route table
entry we are about to make. We will add this configuration in the
`/etc/network/interfaces.d/${INTERFACE}.cfg`. This file will be run when the
interface, `eth1` for us, is attached.

This will get the machine ready by configuring the interface. Now this project
did nearly everything we needed right out of the box but we had to make a few
tweaks which are detailed below.

So we used our own setup script but followed everything that the Makefile
script. When we booted up the instance, everything was configured but the
interface was not up. So we [update the 53-ec2-network-interfaces.rules] with
the following code:

```bash
# 53-ec2-network-interfaces.rules
ACTION=="add", SUBSYSTEM=="net", KERNEL=="eth*", \
  RUN+="/sbin/ifup $env{INTERFACE}"
ACTION=="remove", SUBSYSTEM=="net", KERNEL=="eth*", \
  RUN+="/sbin/ifdown $env{INTERFACE}"
```

This would actually make the network interface available for use. However, we
also needed one more piece for this to actually work. We needed to do a [scan]
to find any unconfigured interfaces.

```bash
# ec2ifscan

#!/bin/bash

# Copyright (C) 2013 Amazon.com, Inc. or its affiliates.
# All Rights Reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License").
# You may not use this file except in compliance with the License.
# A copy of the License is located at
#
#    http://aws.amazon.com/apache2.0/
#
# or in the "license" file accompanying this file. This file is
# distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS
# OF ANY KIND, either express or implied. See the License for the
# specific language governing permissions and limitations under the
# License.

if [ $UID -ne 0 ]; then
  echo "error: ${0##*/} must be run as root"
  exit 1
fi

for dev in $(find /sys/class/net/eth*) ; do
  cfg="/etc/network/interfaces.d/${dev##*/}.cfg"
  state=$(cat ${dev}/operstate)
  if [ ! -e "${cfg}" ] && [ "${state}" == "down" ] ; then
    echo 'add' > ${dev}/uevent
  fi
done
```

The scan code above would not just run on it's own so we monitored the
availability of the eth0 interface (which is configured by default) via an
[upstart script] and then scanning to configure any other elastic network
interfaces that might be attached.

```bash
# elastic-network-interfaces.conf

# This task finds and configures elastic network interfaces
# left in an unconfigured state.

start on net-device-up IFACE=eth0
task
exec /sbin/ec2ifscan
```

In the end this was enough for us to have the network interface configured and
unconfigured on the add and remove `udev` events for that interface. This was
just the pre-requisite to the other problem that we were trying to solution,
allowing docker containers to work over multiple interfaces. This will be the
topic for another blog post which is coming soon.

[friend]: https://github.com/brentonr
[project]: https://github.com/bkr/ubuntu-ec2net
[53-ec2-network-interfaces.rules]: https://github.com/brentonr/ubuntu-ec2net/blob/2a4c30cfb5f60b2acfdbf2d3fe473a32a29490f2/53-ec2-network-interfaces.rules
[ec2net.hotplug]: https://github.com/brentonr/ubuntu-ec2net/blob/2a4c30cfb5f60b2acfdbf2d3fe473a32a29490f2/ec2net.hotplug
[ec2net-functions]: https://github.com/brentonr/ubuntu-ec2net/blob/6a3171f7e5edcebde09d0666a84ccf55b8060de9/ec2net-functions
[update the 53-ec2-network-interfaces.rules]: https://github.com/brentonr/ubuntu-ec2net/commit/6a3171f7e5edcebde09d0666a84ccf55b8060de9
[scan]: https://github.com/brentonr/ubuntu-ec2net/blob/master/ec2ifscan
[upstart script]: https://github.com/brentonr/ubuntu-ec2net/blob/master/elastic-network-interfaces.conf
