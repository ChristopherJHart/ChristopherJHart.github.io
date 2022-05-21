---
layout: post
title: Bridge Network Interfaces on Ubuntu 22.04
---

Linux hosts can bridge two network interfaces together such that they behave link a normal Ethernet link. This means that traffic entering one network interface will exit the other network interface (and vice versa). This can be a useful tool for:

* Performing a packet capture on a link to troubleshoot an issue, especially when a packet capture on either side of a link could be inaccurate.
* Introducing artificial link conditioning between two devices on a link, such as latency or packet loss.

This article will walk through how to bridge two network devices on a Linux host running the Ubuntu 22.04 operating system.

## Topology

First, let's review the topology we will use for this article.

![]({{ site.baseurl }}/images/2022/bridge-network-interfaces-on-ubuntu/topology.jpg)

This article will feature three Linux hosts running Ubuntu 20.04. Hosts "H1" and "H2" are both connected to a host named "Bridge" via interface eth1. H1's eth1 is assigned an IP address of 192.0.2.1/30 with the `ip address add 192.0.2.1/30 dev eth1` command.

```
root@H1:/# ip address add 192.0.2.1/30 dev eth1
root@H1:/# ip address show eth1
113: eth1@if112: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 qdisc noqueue state UP group default
    link/ether aa:c1:ab:a4:88:fb brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 192.0.2.1/30 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a8c1:abff:fea4:88fb/64 scope link
       valid_lft forever preferred_lft forever
```

H2's eth1 is assigned an IP address of 192.0.2.2/30 with the `ip address add 192.0.2.2/30 dev eth1` command.

```
root@H2:/# ip address add 192.0.2.2/30 dev eth1
root@H2:/# ip address show eth1
115: eth1@if114: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 qdisc noqueue state UP group default
    link/ether aa:c1:ab:a1:db:72 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 192.0.2.2/30 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a8c1:abff:fea1:db72/64 scope link
       valid_lft forever preferred_lft forever
```

The Bridge host will bridge its eth1 and eth2 interfaces together such that any packet entering Bridge's eth1 will be forwarded out of Bridge's eth2 interface (and vice versa). The Bridge host will be completely transparent to both H1 and H2; H1 and H2 will both believe that they are directly connected to each other and will not be aware of Bridge's existence.

We can confirm that H1 and H2 do not have connectivity to each other at the moment through ICMP pings sent through the `ping` command.

```
root@H1:/# ping 192.0.2.2 -c 5
PING 192.0.2.2 (192.0.2.2) 56(84) bytes of data.
From 192.0.2.1 icmp_seq=1 Destination Host Unreachable
From 192.0.2.1 icmp_seq=2 Destination Host Unreachable
From 192.0.2.1 icmp_seq=3 Destination Host Unreachable
From 192.0.2.1 icmp_seq=4 Destination Host Unreachable
From 192.0.2.1 icmp_seq=5 Destination Host Unreachable

--- 192.0.2.2 ping statistics ---
5 packets transmitted, 0 received, +5 errors, 100% packet loss, time 4076ms
pipe 4

root@H2:/# ping 192.0.2.1 -c 5
PING 192.0.2.1 (192.0.2.1) 56(84) bytes of data.
From 192.0.2.2 icmp_seq=1 Destination Host Unreachable
From 192.0.2.2 icmp_seq=2 Destination Host Unreachable
From 192.0.2.2 icmp_seq=3 Destination Host Unreachable
From 192.0.2.2 icmp_seq=4 Destination Host Unreachable
From 192.0.2.2 icmp_seq=5 Destination Host Unreachable

--- 192.0.2.1 ping statistics ---
5 packets transmitted, 0 received, +5 errors, 100% packet loss, time 4251ms
pipe 3
```

## Creating a Bridge

First, we will create a network bridge interface named "br0" on the Bridge host with the `ip link add name br0 type bridge` command.

```
root@Bridge:/# ip link add name br0 type bridge
```

By default, the br0 network bridge interface will be administratively down, which means traffic will not flow through the network bridge interface. Therefore, we will bring the br0 network bridge interface up with the `ip link set dev br0 up`.

```
root@Bridge:/# ip link set dev br0 up
```

Finally, we will add our eth1 and eth2 physical interfaces to the br0 network bridge interface with the `ip link set dev eth1 master br0` and `ip link set dev eth2 master br0` commands.

```
root@Bridge:/# ip link set dev eth1 master br0
root@Bridge:/# ip link set dev eth2 master br0
```

At this point, traffic should now begin flowing between the eth1 and eth2 interfaces through the br0 network bridge interface. We can confirm that the br0 network bridge interface is configured as desired with the `bridge link` command.

```
root@Bridge:/# bridge link
114: eth2@if115: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 master br0 state forwarding priority 32 cost 2
112: eth1@if113: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 master br0 state forwarding priority 32 cost 2
```

> **Note**: If you see an error about either the `ip` or `bridge` commands not existing, you will need to install the `iproute2` package with the `sudo apt -y install iproute2` command.

We can also confirm that the br0 interface itself is up with the `ip link show br0` command.

```
root@Bridge:/# ip link show br0
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether aa:c1:ab:6b:33:ae brd ff:ff:ff:ff:ff:ff
```

## Validating Bridge Functionality

Now that the br0 bridge interface is configured and operational, we can confirm bidirectional connectivity between the H1 and H2 hosts through ICMP pings with the `ping` command. As you can see below, both hosts are now able to ping each other through the Bridge host's br0 bridge interface.

```
root@H1:/# ping 192.0.2.2 -c 5
PING 192.0.2.2 (192.0.2.2) 56(84) bytes of data.
64 bytes from 192.0.2.2: icmp_seq=1 ttl=64 time=0.057 ms
64 bytes from 192.0.2.2: icmp_seq=2 ttl=64 time=0.057 ms
64 bytes from 192.0.2.2: icmp_seq=3 ttl=64 time=0.055 ms
64 bytes from 192.0.2.2: icmp_seq=4 ttl=64 time=0.052 ms
64 bytes from 192.0.2.2: icmp_seq=5 ttl=64 time=0.050 ms

--- 192.0.2.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4084ms
rtt min/avg/max/mdev = 0.050/0.054/0.057/0.002 ms
```

```
root@H2:/# ping 192.0.2.1 -c 5
PING 192.0.2.1 (192.0.2.1) 56(84) bytes of data.
64 bytes from 192.0.2.1: icmp_seq=1 ttl=64 time=0.039 ms
64 bytes from 192.0.2.1: icmp_seq=2 ttl=64 time=0.049 ms
64 bytes from 192.0.2.1: icmp_seq=3 ttl=64 time=0.048 ms
64 bytes from 192.0.2.1: icmp_seq=4 ttl=64 time=0.050 ms
64 bytes from 192.0.2.1: icmp_seq=5 ttl=64 time=0.049 ms

--- 192.0.2.1 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4081ms
rtt min/avg/max/mdev = 0.039/0.047/0.050/0.004 ms
```

## Troubleshooting Bridge Connectivity

If you are facing connectivity issues between devices connected through a bridge interface, you can try to resolve them through the below steps.

### Validate Network Bridge Interface Status

Validate that the network bridge interface is administratively up. The output of `ip link show br0` for a bridge interface in an administratively down state will have `state DOWN`, and the output will not have `UP,LOWER_UP` present. An example of an administratively down network bridge interface is below.

```
root@Bridge:/# ip link show br0
2: br0: <BROADCAST,MULTICAST> mtu 9500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000
    link/ether aa:c1:ab:6b:33:ae brd ff:ff:ff:ff:ff:ff
```

An example of an administratively up network bridge interface is below.

```
root@Bridge:/# ip link show br0
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether aa:c1:ab:6b:33:ae brd ff:ff:ff:ff:ff:ff
```

### Validate Physical Network Interface Status

Validate that the physical interfaces assigned to the network bridge interface are administratively up and functional. If at least one of the physical interfaces assigned to a network bridge interface are administratively up and functional, then the network bridge interface status will have `LOWER_UP` present in the output of `ip link show br0`.

An example of this is shown below. The output of the `bridge link` command shows that the eth1 interface is down (due to an absence of `UP,LOWER_UP` in eth1's status). However, the output of the `ip link show br0` command suggests that the network bridge interface is up and operational.

```
root@Bridge:/# bridge link
114: eth2@if115: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 master br0 state forwarding priority 32 cost 2
112: eth1@if113: <BROADCAST,MULTICAST> mtu 9500 master br0 state disabled priority 32 cost 2

root@Bridge:/# ip link show br0
2: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether aa:c1:ab:6b:33:ae brd ff:ff:ff:ff:ff:ff
```

If none of the physical interfaces assigned to a network bridge interface are administratively up and functional, then the network bridge interface status will not have `LOWER_UP` present in the output of `ip link show br0`. Furthermore, the output of `ip link show br0` will have `NO-CARRIER` present, which indicates that the interface is administratively up, but operationally down.

An example of this is shown below. The output of the `bridge link` command shows that both the eth1 and eth2 interfaces are administratively down. The output of the `ip link show br0` command shows that the br0 network bridge interface is administratively up, but operationally down due to the `NO-CARRIER` flag present in the interface's status.

```
root@Bridge:/# bridge link
114: eth2@if115: <BROADCAST,MULTICAST> mtu 9500 master br0 state disabled priority 32 cost 2
112: eth1@if113: <BROADCAST,MULTICAST> mtu 9500 master br0 state disabled priority 32 cost 2

root@Bridge:/# ip link show br0
2: br0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 9500 qdisc noqueue state DOWN mode DEFAULT group default qlen 1000
    link/ether aa:c1:ab:6b:33:ae brd ff:ff:ff:ff:ff:ff
```

### Packet Captures on Network Bridge Interfaces

You can use your packet capture tool of choice (tcpdump, tshark, termshark, etc.) on a network bridge interface to confirm whether traffic is flowing through the network bridge or not. An example of this is shown below, where the `tcpdump` utility is listening for traffic on the br0 network bridge interface while an ICMP ping is executed from H1 to H2.

```
root@Bridge:/# tcpdump -i br0
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on br0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
18:21:21.335916 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 21, seq 1, length 64
18:21:21.335942 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 21, seq 1, length 64
18:21:22.362541 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 21, seq 2, length 64
18:21:22.362569 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 21, seq 2, length 64
18:21:23.386857 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 21, seq 3, length 64
18:21:23.386887 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 21, seq 3, length 64
18:21:24.407200 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 21, seq 4, length 64
18:21:24.407230 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 21, seq 4, length 64
18:21:25.439493 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 21, seq 5, length 64
18:21:25.439526 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 21, seq 5, length 64
```

You may also perform packet captures on physical interfaces allocated to the network bridge. This can be useful to confirm whether traffic from a connected device is entering a specific physical interface allocated to the network bridge or not. An example of this is shown below, where the `tcpdump` utility is listening for traffic on the eth1 physical interface allocated to the br0 network bridge interface while an ICMP ping is executed from H1 to H2.

```
root@Bridge:/# tcpdump -i eth1
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), snapshot length 262144 bytes
18:22:06.590936 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 22, seq 1, length 64
18:22:06.590965 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 22, seq 1, length 64
18:22:07.607107 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 22, seq 2, length 64
18:22:07.607135 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 22, seq 2, length 64
18:22:08.636128 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 22, seq 3, length 64
18:22:08.636159 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 22, seq 3, length 64
18:22:09.657644 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 22, seq 4, length 64
18:22:09.657674 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 22, seq 4, length 64
18:22:10.685826 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 22, seq 5, length 64
18:22:10.685858 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 22, seq 5, length 64
```

Another example of this is shown below, where the `tcpdump` utility is listening for traffic on the eth2 physical interface allocated to the br0 network bridge interface. This confirms that the ICMP ping traffic from H1 to H2 egresses the eth2 physical interface towards H2.

```
root@Bridge:/# tcpdump -i eth2
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on eth2, link-type EN10MB (Ethernet), snapshot length 262144 bytes
18:22:26.959312 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 23, seq 1, length 64
18:22:26.959327 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 23, seq 1, length 64
18:22:27.994407 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 23, seq 2, length 64
18:22:27.994425 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 23, seq 2, length 64
18:22:29.014559 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 23, seq 3, length 64
18:22:29.014575 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 23, seq 3, length 64
18:22:30.038857 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 23, seq 4, length 64
18:22:30.038875 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 23, seq 4, length 64
18:22:31.062862 IP 192.0.2.1 > 192.0.2.2: ICMP echo request, id 23, seq 5, length 64
18:22:31.062879 IP 192.0.2.2 > 192.0.2.1: ICMP echo reply, id 23, seq 5, length 64
```
