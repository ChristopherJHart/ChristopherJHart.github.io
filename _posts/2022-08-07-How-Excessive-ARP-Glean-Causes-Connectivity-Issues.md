---
layout: post
title: How Security Scanners Cause Network Outages - CoPP and ARP Glean in Production Networks
---

> **Note**: If you are not familiar with the concepts of the data plane and control plane on network devices, I highly recommend reviewing my [Understanding the Data, Control, and Management Planes of Network Devices post]({{ site.baseurl }}/Understanding-Data-Control-Management-Planes/) prior to reading this post. If you are not familiar with the concept of ARP Glean, I also recommend reviewing my [Understanding ARP Glean post]({{ site.baseurl }}/Understanding-ARP-Glean/) prior to reading this post.

Most enterprise and data center networks have at least one instance of vulnerability scanning software deployed somewhere in the network. This software regularly scans ranges of IPv4 addresses within the network in an effort to identify active hosts, probe them further, and analyze them for open vulnerabilities. A few examples of such software include Nessus, Burp Suite, Qualys, Nmap/Zenmap, and so on.

A common issue seen in these networks is where scheduled vulnerability scans appear to cause intermittent connectivity issues between hosts. During the time of the scan, you may see intermittent application slowness or have difficulty connecting to hosts via HTTP, SSH, and so on. As soon as the scheduled vulnerability scan finishes or is cancelled, the connectivity issues dissipate, which suggests the vulnerability scan itself may be the cause of the issue.

In this blog post, we will reproduce this behavior and demonstrate how the delicate dance between Control Plane Policing (CoPP) and ARP Glean can cause these types of connectivity issues in a production network.

## Topology

First, let's review the topology we will use in this article.

![]({{ site.baseurl }}/images/2022/arp-glean-connectivity-issues/ARP Glean Blog Post Topology.jpg)

In this topology, we have two Cisco Nexus 93360YC-FX2 switches running NX-OS software release 10.2(3) configured in a vPC domain. Ethernet1/48 of both switches connects to separate ports of a Keysight Ixia traffic generator; Ethernet1/48 of N9K-1 connects to Card 3 Port 15 of the Ixia, while Ethernet1/48 of N9K-2 connects to Card 3 Port 16 of the Ixia. Ethernet1/47 of N9K-1 is connected to Card 3 Port 14 of the Ixia as well.

This topology has a total of eleven VLANs - VLANs 1-10, and VLAN 100. N9K-1 and N9K-2 are running HSRP (Hot Standby Router Protocol) and are both acting as gateways for each VLAN (as both vPC peers can act as gateways for vPC VLANs). Card 3 Port 15 of the Ixia emulates hosts in VLAN1-10, with one host per VLAN. Each host has an IP address ending in .10 (so 192.168.1.10 in VLAN 1, 192.168.2.10 in VLAN 2, etc.) Card 3 Port 16 of the Ixia emulates hosts identically, except each host has an IP address ending in .20 (so 192.168.1.20 in VLAN 1, 192.168.2.20 in VLAN 2, etc.). This emulates legitimate hosts in our production network, and each port sends traffic in a full-mesh manner such that our emulated production network has both intra-VLAN and inter-VLAN traffic flowing through both Nexus switches.

Card 3 Port 14 of the Ixia traffic generator emulates a security scanner. This port will send UDP packets towards random IP addresses in the 10.0.0.0/16 subnet associated with VLAN 100 sourced from 192.168.1.25 in VLAN 1. These packets are destined to the HSRP virtual MAC address, which means N9K-1 must route this traffic from VLAN 1 into VLAN 100. These packets will be treated as ARP Glean and will be punted to the control plane, as N9K-1 does not have ARP resolved for any IP address in the 10.0.0.0/16 subnet.

## Working State Traffic Statistics

First, let's analyze this topology in the "working state", when the traffic from the Ixia's Card 3 Port 14 emulating the security scanner is not running. In this steady state, no loss is observed for the production traffic running between the other two ports of the Ixia traffic generator. The 22 frames that Ixia reports are "missing" are the frames in-flight between a transmitting port and a receiving port - this is not indicative of packet loss, as the value is not steadily incrementing.

![]({{ site.baseurl }}/images/2022/arp-glean-connectivity-issues/glean_working_state_no_packet_loss.jpg)

Next, let's see how much packet loss is incurred as a result of ARP Glean and CoPP when a single switch's ARP table is completely wiped. First, we will clear CoPP statistics on N9K-1 with the `clear copp statistics` command.

```
N9K-1# clear copp statistics
N9K-1# 
```

Next, let's validate that no packets have been dropped under any CoPP class.

```
N9K-1# show policy-map interface control-plane | include class|dropped | exclude dropped.0.bytes
    class-map copp-system-p-class-l3uc-data (match-any)
    class-map copp-system-p-class-critical (match-any)
    class-map copp-system-p-class-important (match-any)
    class-map copp-system-p-class-openflow (match-any)
    class-map copp-system-p-class-multicast-router (match-any)
    class-map copp-system-p-class-multicast-host (match-any)
    class-map copp-system-p-class-l3mc-data (match-any)
    class-map copp-system-p-class-normal (match-any)
    class-map copp-system-p-class-ndp (match-any)
    class-map copp-system-p-class-normal-dhcp (match-any)
    class-map copp-system-p-class-normal-dhcp-relay-response (match-any)
    class-map copp-system-p-class-normal-igmp (match-any)
    class-map copp-system-p-class-redirect (match-any)
    class-map copp-system-p-class-exception (match-any)
    class-map copp-system-p-class-exception-diag (match-any)
    class-map copp-system-p-class-management (match-any)
    class-map copp-system-p-class-monitoring (match-any)
    class-map copp-system-p-class-l2-unpoliced (match-any)
    class-map copp-system-p-class-undesirable (match-any)
    class-map copp-system-p-class-fcoe (match-any)
    class-map copp-system-p-class-nat-flow (match-any)
    class-map copp-system-p-class-l3mcv6-data (match-any)
    class-map copp-system-p-class-undesirablev6 (match-any)
    class-map copp-system-p-class-l2-default (match-any)
    class-map class-default (match-any)
```

Finally, let's clear N9K-1's ARP table with the `clear ip arp force-delete` command. The `force-delete` keyword deletes an entry in the switch's ARP table *without* attempting to re-resolve ARP immediately afterwards.

```
N9K-1# clear ip arp force-delete
N9K-1# 
```

If we reference our Ixia traffic statistics, we can see that about 22,777 frames were lost. Note that the number of frames transmitted/received is about 452,900 frames per second, which means the total outage time was about 50 milliseconds (22,777 / 452,900 = 0.0503 seconds) in length.

![]({{ site.baseurl }}/images/2022/arp-glean-connectivity-issues/glean_working_state_some_packet_loss.jpg)

If we check CoPP statistics, we can see 5,798,784 bytes were dropped under the "copp-system-p-class-l3uc-data" CoPP class map.

```
N9K-1# show policy-map interface control-plane | include class|dropped | exclude dropped.0.bytes
    class-map copp-system-p-class-l3uc-data (match-any)
        dropped 5798784 bytes;
    class-map copp-system-p-class-critical (match-any)
    class-map copp-system-p-class-important (match-any)
    class-map copp-system-p-class-openflow (match-any)
    class-map copp-system-p-class-multicast-router (match-any)
    class-map copp-system-p-class-multicast-host (match-any)
    class-map copp-system-p-class-l3mc-data (match-any)
    class-map copp-system-p-class-normal (match-any)
    class-map copp-system-p-class-ndp (match-any)
    class-map copp-system-p-class-normal-dhcp (match-any)
    class-map copp-system-p-class-normal-dhcp-relay-response (match-any)
    class-map copp-system-p-class-normal-igmp (match-any)
    class-map copp-system-p-class-redirect (match-any)
    class-map copp-system-p-class-exception (match-any)
    class-map copp-system-p-class-exception-diag (match-any)
    class-map copp-system-p-class-management (match-any)
    class-map copp-system-p-class-monitoring (match-any)
    class-map copp-system-p-class-l2-unpoliced (match-any)
    class-map copp-system-p-class-undesirable (match-any)
    class-map copp-system-p-class-fcoe (match-any)
    class-map copp-system-p-class-nat-flow (match-any)
    class-map copp-system-p-class-l3mcv6-data (match-any)
    class-map copp-system-p-class-undesirablev6 (match-any)
    class-map copp-system-p-class-l2-default (match-any)
    class-map class-default (match-any)
```

If we investigate this class map further, we can see that glean traffic is matched under this class through the "match exception glean" clause under the relevant class map.

```
N9K-1# show policy-map interface control-plane class copp-system-p-class-l3uc-data
Control Plane

  Service-policy  input: copp-system-p-policy-strict

    class-map copp-system-p-class-l3uc-data (match-any)
      match exception glean
      set cos 1
      police cir 800 kbps , bc 32000 bytes 
      module 1 : 
        transmitted 44100 bytes;
        5-minute offered rate 51 bytes/sec
        conformed 6300 peak-rate bytes/sec
          at Sun Aug 07 21:16:25 2022

        dropped 5798784 bytes;
        5-min violate rate 6709 byte/sec
        violated 828397 peak-rate byte/sec        at Sun Aug 07 21:16:25 2022
```

The purpose of this test is to show the minimal impact of a *typical* interaction with ARP Glean traffic. Usually, if a router receives a large amount of traffic destined to one or more new hosts (where "new" means that the host's IPv4 address was not resolved to a MAC address in the router's ARP table), traffic to those hosts is dropped for a few milliseconds as it gets punted to the control plane for resolution. Once the host's MAC address is resolved via ARP, the traffic is forwarded normally through the data plane.

Now that we understand what a typical ARP Glean interaction looks like, let's explore the impact a security scanner can have on a router's ability to maintain its ARP table.

## Broken State Traffic Statistics

After clearing CoPP statistics and Ixia traffic statistics once more, we instructed Card 3 Port 14 of the Ixia to begin sending UDP traffic destined to random hosts in the 10.0.0.0/16 subnet through N9K-1. This traffic will be sent at a rate of 1Mbps. Note that the existing production traffic has not been stopped for this test.

The CoPP statistics for N9K-1 show that N9K-1 is receiving a large amount of ARP Glean traffic, which is being dropped in hardware.

```
N9K-1# show policy-map interface control-plane class copp-system-p-class-l3uc-data
Control Plane

  Service-policy  input: copp-system-p-policy-strict

    class-map copp-system-p-class-l3uc-data (match-any)
      match exception glean
      set cos 1
      police cir 800 kbps , bc 32000 bytes 
      module 1 : 
        transmitted 30448872 bytes;
        5-minute offered rate 96939 bytes/sec
        conformed 99684 peak-rate bytes/sec
          at Sun Aug 07 21:41:20 2022

        dropped 323259196 bytes;
        5-min violate rate 800718 byte/sec
        violated 1102201 peak-rate byte/sec        at Sun Aug 07 21:46:23 2022
```

If we check on the Ixia traffic statistics for our production traffic, we can see a fair amount of packet loss has crept into our production network.

![]({{ site.baseurl }}/images/2022/arp-glean-connectivity-issues/glean_broken_state_some_packet_loss.jpg)

At first glance, this may not make sense. Both Nexus switches had a populated ARP table *before* the security scanner traffic started. When an ARP entry gets close to expiring, the Nexus should attempt to proactively refresh that entry by sending an ARP request for that IP to confirm the same IP/MAC binding is still valid. This traffic is ARP traffic, which should fall under the "copp-system-p-class-normal" CoPP class-map, not the Glean-focused "copp-system-p-class-l3uc-data".

```
N9K-1# show policy-map interface control-plane class copp-system-p-class-normal
Control Plane

  Service-policy  input: copp-system-p-policy-strict

    class-map copp-system-p-class-normal (match-any)
      match access-group name copp-system-p-acl-mac-dot1x
      match protocol arp
      set cos 1
      police cir 1400 kbps , bc 32000 bytes 
      module 1 : 
        transmitted 162544 bytes;
        5-minute offered rate 82 bytes/sec
        conformed 324 peak-rate bytes/sec
          at Sun Aug 07 21:46:31 2022

        dropped 0 bytes;
        5-min violate rate 0 byte/sec
        violated 0 peak-rate byte/sec
```

## Analysis

So if the ARP table should remain stable, where is the packet loss coming from? Let's briefly turn our attention to N9K-2's CoPP statistics.

```
N9K-2# show policy-map interface control-plane | include class|dropped | exclude dropped.0.bytes
    class-map copp-system-p-class-l3uc-data (match-any)
        dropped 6042828 bytes;
    class-map copp-system-p-class-critical (match-any)
    class-map copp-system-p-class-important (match-any)
    class-map copp-system-p-class-openflow (match-any)
    class-map copp-system-p-class-multicast-router (match-any)
    class-map copp-system-p-class-multicast-host (match-any)
    class-map copp-system-p-class-l3mc-data (match-any)
    class-map copp-system-p-class-normal (match-any)
        dropped 485425772 bytes;
    class-map copp-system-p-class-ndp (match-any)
    class-map copp-system-p-class-normal-dhcp (match-any)
    class-map copp-system-p-class-normal-dhcp-relay-response (match-any)
    class-map copp-system-p-class-normal-igmp (match-any)
    class-map copp-system-p-class-redirect (match-any)
    class-map copp-system-p-class-exception (match-any)
    class-map copp-system-p-class-exception-diag (match-any)
    class-map copp-system-p-class-management (match-any)
    class-map copp-system-p-class-monitoring (match-any)
    class-map copp-system-p-class-l2-unpoliced (match-any)
    class-map copp-system-p-class-undesirable (match-any)
    class-map copp-system-p-class-fcoe (match-any)
    class-map copp-system-p-class-nat-flow (match-any)
    class-map copp-system-p-class-l3mcv6-data (match-any)
    class-map copp-system-p-class-undesirablev6 (match-any)
    class-map copp-system-p-class-l2-default (match-any)
    class-map class-default (match-any)
```

Here, we can see drops under N9K-2's "copp-system-p-class-normal" CoPP class-map, which is where ARP traffic would fall under. Where is this ARP traffic coming from? A brief look at the NX-OS Ethanalyzer control plane packet capture utility should tell us.

```
N9K-2# ethanalyzer local interface inband display-filter arp limit-captured-frames 0
Capturing on 'ps-inb'
    1 2022-08-07 21:57:00.494428263 90:77:ee:36:4f:ab ‚Üí ff:ff:ff:ff:ff:ff ARP 64 Who has 10.0.26.87? Tell 10.0.0.2
    2 2022-08-07 21:57:00.494828265 90:77:ee:36:4f:ab ‚Üí ff:ff:ff:ff:ff:ff ARP 64 Who has 10.0.236.145? Tell 10.0.0.2
    3 2022-08-07 21:57:00.495502114 90:77:ee:36:4f:ab ‚Üí ff:ff:ff:ff:ff:ff ARP 64 Who has 10.0.11.23? Tell 10.0.0.2
    4 2022-08-07 21:57:00.495890139 90:77:ee:36:4f:ab ‚Üí ff:ff:ff:ff:ff:ff ARP 64 Who has 10.0.26.90? Tell 10.0.0.2
    5 2022-08-07 21:57:00.496387868 90:77:ee:36:4f:ab ‚Üí ff:ff:ff:ff:ff:ff ARP 64 Who has 10.0.236.148? Tell 10.0.0.2
    6 2022-08-07 21:57:00.497446750 90:77:ee:36:4f:ab ‚Üí ff:ff:ff:ff:ff:ff ARP 64 Who has 10.0.26.93? Tell 10.0.0.2
    7 2022-08-07 21:57:00.498057860 90:77:ee:36:4f:ab ‚Üí ff:ff:ff:ff:ff:ff ARP 64 Who has 10.0.11.28? Tell 10.0.0.2
    8 2022-08-07 21:57:00.498446907 90:77:ee:36:4f:ab ‚Üí ff:ff:ff:ff:ff:ff ARP 64 Who has 10.0.26.95? Tell 10.0.0.2
```

Here, we can see that N9K-1 (who owns IP 10.0.0.2) is generating a large amount of ARP Requests for a variety of IPv4 addresses in the 10.0.0.0/16 subnet. The sequence of events causing this outage is beginning to take shape!

1. The security scanner is sending a large amount of traffic to a variety of IPv4 addresses in the 10.0.0.0/16 subnet.
2. N9K-1 receives this traffic and punts it to the control plane, attempting to use ARP to resolve a MAC address for each IPv4 address.
3. Since N9K-1 is inundated by ARP Glean traffic, it logically generates a large amount of ARP Requests.
4. N9K-2's control plane is overwhelmed by these ARP Requests and has no choice but to start dropping them in hardware due to CoPP.
5. When N9K-2 needs to refresh an entry in its ARP table, it sends an ARP Request out for the relevant IPv4 address. The ARP Reply in response is "starved out" by the large quantity of ARP Requests sent by N9K-1.
6. Eventually, N9K-2's ARP entry expires, causing production traffic destined to the IPv4 address in that entry to be punted to the control plane as ARP Glean. This action (combined with N9K-2's inability to expeditiously resolve ARP for the production host's IPv4 address) introduces our connectivity issues, which could include throughput issues, application performance issues, and outright connectivity issues.

Statistically, *eventually* the ARP Reply for the production host will be allowed by N9K-2's CoPP policy, allowing N9K-2 to resolve ARP for that host. This adds complexity to the problem. First, the *occurrence* of the problem seems extremely intermittent - sometimes an individual host or flow of traffic will stop working and then start working again. Second, the *impact* of the problem seems extremely intermittent - the individual host or flow affected by the issue will change from one instance of the issue to another.

Ultimately, the fix for this issue is one of two actions:

1. Reduce the "aggressiveness" of the security scanner by configuring it to scan subnets "slower" such that the bandwidth of traffic does not overwhelm the CoPP policy of core switches. This is the "correct" action in the overwhelming majority of cases.
2. Modify the CoPP policy of core switches such that rate limit of the ARP Glean or ARP class maps (or both) is increased. This *can* be the correct action in some cases where business requirements dictate (e.g. "Due to compliance, we must scan the network for vulnerable hosts every 4 hours, no matter what, and reducing the scan speed will break our compliance).

> **Note**: The correct action is ***never*** to disable CoPP entirely on your switches. Not only is this not supported on Nexus 9000 series switches, but logically, this is equivalent to driving without a seat belt.

## Conclusion

In this article, we demonstrated a practical, real-world example of vulnerability security scanners can sometimes wreak havoc in a production network in odd, unpredictable ways. By combining our understanding of network device control and data planes, our knowledge of ARP Glean, and our comprehension of CoPP, we can identify the sequence of events that causes vulnerability security scanners to cause network degradation or outages and take the appropriate steps to fix them.

## Related Reading

* [Understanding the Data, Control, and Management Planes of Network Devices]({{ site.baseurl }}/Understanding-Data-Control-Management-Planes/)
* [Understanding Control Plane Packet Loss due to CoPP]({{ site.baseurl }}/Understanding-Control-Plane-Packet-Loss-due-to-CoPP/)
* [ICMP Redirects - How Data Plane Traffic Can Become Control Plane Traffic]({{ site.baseurl }}/How-Data-Plane-Traffic-Can-Become-Control-Plane-Traffic/)
* [Understanding ARP Glean]({{ site.baseurl }}/Understanding-ARP-Glean/)
