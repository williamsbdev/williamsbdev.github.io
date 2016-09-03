---
layout: post
title: "Java and TCP status CLOSE_WAIT"
tags: [devops, java]
---

In a [previous post], I talked briefly about the TCP status [CLOSE_WAIT]. The
problem occurs between two nodes (we will refer to the node initiating the
connection as the client and the one receiving the connection the server) when
the server initiates a shutdown and closes the connection. The client however,
has not closed the connection (socket) so now the connection on the client's
side is only half open.

With that review of the problem, we will now dive into the
troubleshooting/debugging process. On my local machine, I was running two
services that would be communicating (client and server for our example). The
server was started and happily serving requests from the client. Since I had
the HTTP connection pooling in the client that I described in my previous post,
there were long-lived connections to reduce the overhead of creating a new
connection on each request. In order to establish the baseline and see what
things looked like in a happy scenario I did the following:

```bash
ps aux | grep client.jar
```

which produces the following output:

```bash
USER       PID %CPU %MEM     VSZ    RSS TTY    STAT START   TIME COMMAND
root     28559 34.4  1.0 4702560 348888 ?      Ssl  20:22   1:16 /jre/bin/java -jar /apps/client.jar
```

This would find me the process ID that was running (which is actually a process
started in a docker container that I can see on the host system). Now we will
the following:

```bash
cd /proc/28559/net/
ls | grep tcp
```

This will show us the following:

```bash
tcp
tcp6
```

You will notice that there is a `tcp` and a `tcp6` file. Depending on how the
process was binding to localhost, the connection information would be in the
corresponding file where `tcp` is the file for IPv4 connections/bindings and
`tcp6` is for IPv6 connections/bindings. Next we will look for the single
connection that we care about with the following command:

```bash
watch -n 0.001 "grep 1F90 tcp6"
```

The watch command will simply run the `grep 1F90` command every 0.001 seconds
(`-n` option) and output to stdout. This will show the output below:

```bash
sl  local_address                         remote_address                        st tx_queue rx_queue tr tm->when retrnsmt   uid  timeout inode
 5: 0000000000000000FFFF00000C0011AC:9CC0 0000000000000000FFFF0000120011AC:1F90 01 00000000:00000001 00:00000000 00000000     0        0 4648852 1 0000000000000000 20 4 30 10 -1
```

What we are concerned about here is the `01` in the `st` column. The
values in the column can be mapped to the Linux kernel [TCP states]. This
indicates that the connection is `TCP_ESTABLISHED` (the first enum in the
list). Now if we continue to watch this output, we will see that if we were to
terminate the `server.jar` pid, the connection will go from `01` to `08`.

This simulates what happens when servers are being terminated and sending the
shutdown signals to the applications running on them in a real life
environment. The server will initiate it's shutdown sequence, close all sockets
it has open and send a message saying "I'm shutting down and you should too!".
It was at this point that we learned why we would get so many of the Java
`NoHttpResponseException` errors and we started seeking out a way to prevent
them.

[previous post]: https://williamsbdev.com/posts/no-http-response-exceptions
[CLOSE_WAIT]: https://www.google.com/webhp?sourceid=chrome-instant&ion=1&espv=2&ie=UTF-8#q=TCP+status+close_wait
[TCP states]: http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/include/net/tcp_states.h?id=HEAD
