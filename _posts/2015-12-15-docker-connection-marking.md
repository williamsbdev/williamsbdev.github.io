---
layout: post
title: "Docker over multiple network interfaces"
tags: [aws, devops, docker]
---

So in my [previous post], we walked through how you would configure and bring up
a new interface on an AWS Ubuntu EC2 instance. This was just the pre-requisite
to the problem that will be addressed in this post.

We were running a docker container on an instance that had an `eth0` and an
`eth1` interface. Everything was working fine with the software that was
installed on the host level communicating over either IP. However, as soon as I
tried to communicate with a Docker container over the `eth1` (non default
interface), I began to have issues. This is because of what Docker does at the
[ip tables level].

Now this is completely acceptable and needed however, what we were seeing was
that when the packets would come in over `eth1` they would be appropriately
routed through the `docker0` interface to the correct docker container
listening on that port (let's just 4000 for our example). The docker container
application would service the request and attempt to respond back out the
`docker0` interface. Since it lost the source IP (our `eth1` IP) it would
assume the default interface and therefore use the default route table which
would attempt to respond using the `eth0` IP. From the requester's point of
view, there would never be a response.

We solved this by adding to the work we had already done to configure and ready
another network interface. At a high level, we added a FW mark (firewall mark)
to any connection coming in over the `eth1` interface so that as the `docker0`
serviced the request and was about to send it back out, it would restore the
connection mark. Then we have a rule that would run to say, if there is a
connection mark, it is to use the `eth1` route table to figure out which IP to
respond with.

We added another [upstart script] that would do the `iptables` commands we
needed when we knew the `docker0` interface would be available. It would simple
add/remove to/from the `mangle` table in the `PREROUTING` section for any
connection coming from the `docker0` interface to restore any connection marks
that might have been set. The script would then also remove the rule if the
docker service was stopped (machine rebooting or shutting down).

```bash
# etc/init/docker-finish.conf

# docker-finish - update routing tables after docker is up

description "update routing tables after docker is up"

start on started docker
stop on stopped docker

pre-start script
  /sbin/modprobe nf_conntrack
  /sbin/iptables -w -t mangle -A PREROUTING -i docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j CONNMARK --restore-mark --nfmask 0xffffffff --ctmask 0xffffffff
end script

post-stop script
  /sbin/iptables -t mangle -D PREROUTING -i docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j CONNMARK --restore-mark --nfmask 0xffffffff --ctmask 0xffffffff
end script
```

Next, when we were configuring our additional interface, `eth1` in our example,
we would also add the [connection settings] to set the mark. Please see the
code below:

```bash
# excerpt from ec2net-functions

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
  metric ${RTABLE}
post-up ip route add default via ${gateway} dev ${INTERFACE} table ${RTABLE} || true
# use fwmark for connections with masquerading (docker support)
post-up iptables -t mangle -A PREROUTING -i ${INTERFACE} -j MARK --set-xmark 0x${RTABLE}/0xffffffff
post-up iptables -t mangle -A PREROUTING -i ${INTERFACE} -j CONNMARK --save-mark --nfmask 0xffffffff --ctmask 0xffffffff
post-up sysctl -wq net.ipv4.conf.${INTERFACE}.rp_filter=2
post-up bash -c 'bash -c '"'"'until [ "\$(ip route show dev docker0 2>/dev/null)" != "" ]; do sleep 1; done; ip route add \$(ip route show dev docker0) dev docker0 table ${RTABLE}'"'"' &'
pre-down bash -c 'ip route del \$(ip route show dev docker0) dev docker0 table ${RTABLE}' || true
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

Now the [rule] that was added, would say that for any connection with a FW mark
that we set coming in from `eth1`, it would use the new route table that we
configured for the `eth1` interface.

```bash
# excerpt from ec2net-functions line 230

/sbin/ip rule add from all fwmark 0x${RTABLE} lookup ${RTABLE}
```

With these tweaks, we now have the ability to communicate over either `eth0` or
`eth1` to docker containers running on an EC2 instance.

[previous post]: http://williamsbdev.com/posts/hotplug-aws-network-interfaces/
[ip tables level]: https://docs.docker.com/v1.5/articles/networking/
[upstart script]: https://github.com/brentonr/ubuntu-ec2net/blob/b3062f5a6cf2c6339be88b6398ac9fbffc10a731/docker-finish.conf
[connection settings]: https://github.com/brentonr/ubuntu-ec2net/blob/b3062f5a6cf2c6339be88b6398ac9fbffc10a731/ec2net-functions#L115-L120
[rule]: https://github.com/brentonr/ubuntu-ec2net/blob/b3062f5a6cf2c6339be88b6398ac9fbffc10a731/ec2net-functions#L230
