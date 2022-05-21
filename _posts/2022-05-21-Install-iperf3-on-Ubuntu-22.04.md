---
layout: post
title: Installing iPerf3 on Ubuntu 22.04
---

iPerf3 is a popular network performance measurement and testing tool that can be easily installed on most Linux operating systems. This article will walk through how to install iPerf3 on Linux hosts running the Ubuntu 22.04 operating system. It will also show brief examples of how to start the iPerf3 server (for receiving traffic) and how to start the iPerf3 client (for initiating network performance testing towards a host running the iPerf3 server).

## Installing iPerf3

First, use the `sudo apt -y update` command to update the apt package manager's list of packages that can be installed or upgraded.

```
sudo apt -y update
```

Next, install iPerf3 using the `sudo apt -y install iperf3` command.

```
sudo apt -y install iperf3
```

You can confirm that iPerf3 is installed with the `iperf3 -v` command, which will display information about the current version of iPerf3 installed and the current Linux kernel version.

```
iperf3 -v
```

An example of the output of this command is shown below.

```
root@H1:/# iperf3 -v
iperf 3.9 (cJSON 1.7.13)
Linux H1 5.4.0-109-generic #123-Ubuntu SMP Fri Apr 8 09:10:54 UTC 2022 x86_64
Optional features available: CPU affinity setting, IPv6 flow label, SCTP, TCP congestion algorithm setting, sendfile / zerocopy, socket pacing, authentication
```

## Starting an iPerf3 Server

You can confirm that an Ubuntu 22.04 host is able to act as an iPerf3 server with the `iperf3 -s` command.

```
iperf3 -s
```

An example of the output of this command is shown below. If the output of this command indicates that the iPerf3 server is listening on a specific port, then the iPerf3 server is working as expected.

```
root@H2:/# iperf3 -s
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```

By default, the iPerf3 server will listen for new connections on all active interfaces of the host. If you would like the iPerf3 server to listen on a specific interface, you can use the `iperf3 -s -B <ip-address>` command. An example of this is shown below, where the iPerf3 server is instructed to listen on IP address 192.0.2.2.

```
root@H2:/# iperf3 -s -B 192.0.2.2
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```

## Starting an iPerf3 Client

You can confirm that an Ubuntu 22.04 host is able to act as an iPerf3 client with the `iperf3 -c <server-ip-address>` command.

```
iperf3 -c <server-ip-address>
```

An example of this is shown below, where the IP address of the server the iPerf3 client needs to connect to is 192.0.2.2.

```
root@H1:/# iperf3 -c 192.0.2.2
Connecting to host 192.0.2.2, port 5201
[  5] local 192.0.2.1 port 35946 connected to 192.0.2.2 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  3.87 GBytes  33.2 Gbits/sec   57    692 KBytes
[  5]   1.00-2.00   sec  3.63 GBytes  31.2 Gbits/sec    0    812 KBytes
[  5]   2.00-3.00   sec  3.69 GBytes  31.7 Gbits/sec    0    923 KBytes
[  5]   3.00-4.00   sec  3.72 GBytes  32.0 Gbits/sec    0   1.00 MBytes
[  5]   4.00-5.00   sec  3.65 GBytes  31.3 Gbits/sec    0   1.00 MBytes
[  5]   5.00-6.00   sec  4.12 GBytes  35.4 Gbits/sec    0   1.00 MBytes
[  5]   6.00-7.00   sec  3.60 GBytes  30.9 Gbits/sec    0   1.00 MBytes
[  5]   7.00-8.00   sec  3.68 GBytes  31.6 Gbits/sec    0   1.09 MBytes
[  5]   8.00-9.00   sec  4.02 GBytes  34.5 Gbits/sec    0   1.12 MBytes
[  5]   9.00-10.00  sec  3.84 GBytes  33.0 Gbits/sec    0   1.31 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  37.8 GBytes  32.5 Gbits/sec   57             sender
[  5]   0.00-10.00  sec  37.8 GBytes  32.5 Gbits/sec                  receiver

iperf Done.
```

When an iPerf3 client targets a specific iPerf3 server, you will see output similar to the following on the iPerf3 server.

```
root@H2:/# iperf3 -s -B 192.0.2.2
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 192.0.2.1, port 35944
[  5] local 192.0.2.2 port 5201 connected to 192.0.2.1 port 35946
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  3.87 GBytes  33.2 Gbits/sec
[  5]   1.00-2.00   sec  3.63 GBytes  31.2 Gbits/sec
[  5]   2.00-3.00   sec  3.69 GBytes  31.7 Gbits/sec
[  5]   3.00-4.00   sec  3.72 GBytes  32.0 Gbits/sec
[  5]   4.00-5.00   sec  3.65 GBytes  31.3 Gbits/sec
[  5]   5.00-6.00   sec  4.12 GBytes  35.4 Gbits/sec
[  5]   6.00-7.00   sec  3.60 GBytes  30.9 Gbits/sec
[  5]   7.00-8.00   sec  3.68 GBytes  31.6 Gbits/sec
[  5]   8.00-9.00   sec  4.02 GBytes  34.5 Gbits/sec
[  5]   9.00-10.00  sec  3.84 GBytes  33.0 Gbits/sec
[  5]  10.00-10.00  sec   392 KBytes  25.1 Gbits/sec
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-10.00  sec  37.8 GBytes  32.5 Gbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
```

This output indicates that the iPerf3 server successfully accepted a connection from an iPerf3 client and was able to exchange data with the client.
