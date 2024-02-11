---
layout: post
title: "Filtering NX-OS Ethanalyzer on CoPP Classes"
---

Keeping the control plane of a network device free from excessive traffic is a critical component of keeping the network stable and secure. On Cisco Nexus switches, the Control Plane Policing (CoPP) feature protects the control plane of the switch from being overwhelmed by dropping excessive amounts of control plane traffic in hardware. Identifying drops in a switch's CoPP policy can be an excellent "smell test" for misconfigured network devices, applications, or general network stability issues.

If you are not familiar with the concepts of the data plane and control plane on network devices, or are not familiar with how CoPP protects the control plane and can cause create symptoms of an unstable network, I highly recommend reviewing the following posts in order:

* [Understanding the Data, Control, and Management Planes of Network Devices]({{ site.baseurl }}/Understanding-Data-Control-Management-Planes/)
* [Understanding Control Plane Packet Loss due to CoPP]({{ site.baseurl }}/Understanding-Control-Plane-Packet-Loss-due-to-CoPP/)
* [ICMP Redirects - How Data Plane Traffic Can Become Control Plane Traffic]({{ site.baseurl }}/How-Data-Plane-Traffic-Can-Become-Control-Plane-Traffic/)
* [Understanding ARP Glean]({{ site.baseurl }}/Understanding-ARP-Glean/)
* [How Excessive ARP Glean Causes Connectivity Issues]({{ site.baseurl }}/How-Excessive-ARP-Glean-Causes-Connectivity-Issues/)

Troubleshooting drops in a CoPP policy can be a bit challenging in some production environments where the network is very busy. A common example is a core switch performing inter-VLAN routing with a large number of VLANs (and therefore, SVIs), a large number of hosts, and a large number of First Hop Redundancy Protocols (FHRPs, like HSRP or VRRP). In these environments, the control plane is naturally very noisy, which can make it difficult to identify the source of traffic that may be causing CoPP drops within a specific class.

Starting with NX-OS software release 10.1(1), the [Ethanalyzer control plane packet capture utility](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/104x/troubleshooting/cisco-nexus-9000-series-nx-os-troubleshooting-guide-104x/m-troubleshooting-tools-and-methodology.html#reference_khl_lns_nyb) can filter on traffic that matches a specific CoPP class.

> **Note**: The ability to filter Ethanalyzer on a specific CoPP class is only available on Cisco Nexus switches or line cards with the Cisco Cloud Scale ASIC. This typically includes model numbers that end in -EX, -FX, -FX2, -FX3, -GX, -GX2, and so on. This feature is not available on older first-generation Nexus 9000 line cards or switches (which use the Broadcom Trident 2 ASIC, model numbers typically end in TX or PX) or on Nexus 9800 series switches (which use the Cisco Silicon One ASIC).

## Procedure

Let's walk through the steps to set up this filter.

### Identify CoPP Class of Interest

In a hurry? Simply follow the steps below.

First, validate which CoPP class has incrementing drops through the `show policy-map interface control-plane` command. This output can be lengthy, so best practice is to filter the output to only show the class names and their respective dropped byte counts (assuming they're non-zero) with the below command:

```
show policy-map interface control-plane | include class|dropped | exclude dropped.0.bytes
```

Sample filtered output of this command is shown below:

```
switch# show policy-map interface control-plane | include class|dropped | exclude dropped.0.bytes
    class-map copp-system-p-class-l3uc-data (match-any)
    class-map copp-system-p-class-critical (match-any)
    class-map copp-system-p-class-important (match-any)
    class-map copp-system-p-class-openflow (match-any)
    class-map copp-system-p-class-multicast-router (match-any)
    class-map copp-system-p-class-multicast-host (match-any)
    class-map copp-system-p-class-l3mc-data (match-any)
    class-map copp-system-p-class-normal (match-any)
    class-map copp-system-p-class-ndp (match-any)
        dropped 1061714 bytes;
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

In the above output, our CoPP class of interest is `copp-system-p-class-ndp`. As the name suggests, this class is responsible for handling IPv6 Neighbor Discovery Protocol (NDP) traffic. This class has incrementing drops, which we'd like to troubleshoot further.

### Identify CoPP Class Queue Number

Next, execute the `show system internal access-list copp stats stage1` command. Find the entry for the class name corresponding to the class with incrementing drops that you'd like to troubleshoot. Identify the queue number that corresponds with the class name.

```
switch# show system internal access-list copp stats stage1

slot  1
=======


------------------------------------------------------------------------
            COPP Queue Stats Information        
------------------------------------------------------------------------

Ingress: Instance: 0
==========
Queue                              Name               Transmitted(bytes)          Dropped(bytes)
------------------------------------------------------------------------
 1                               Multicast (*,G)                        0                        0
 3                             Exception Default                        0                        0
 4                                      LC OTHER                        0                        0
 5                                         Glean                        0                        0
 6                                          SPAN                        0                        0
 7                                         SFLOW                        0                        0
13                copp-system-p-class-l2-default                        0                        0
14             copp-system-p-class-undesirablev6                        0                        0
15               copp-system-p-class-l3mcv6-data                        0                        0
49                  copp-system-p-class-nat-flow                        0                        0
17                      copp-system-p-class-fcoe                        0                        0
18               copp-system-p-class-undesirable                        0                        0
19              copp-system-p-class-l2-unpoliced                532619153                        0
20                copp-system-p-class-monitoring                 18997192                        0
21                copp-system-p-class-management                        0                        0
22            copp-system-p-class-exception-diag                        0                        0
23                 copp-system-p-class-exception                        0                        0
24                  copp-system-p-class-redirect                        0                        0
25               copp-system-p-class-normal-igmp                  5813790                        0
26copp-system-p-class-normal-dhcp-relay-response                        0                        0
27               copp-system-p-class-normal-dhcp                        0                        0
28                       copp-system-p-class-ndp                 11510328                   127546
29                    copp-system-p-class-normal                  1996136                        0
30                 copp-system-p-class-l3mc-data                        0                        0
31            copp-system-p-class-multicast-host                        0                        0
32          copp-system-p-class-multicast-router                  6665864                        0
33                  copp-system-p-class-openflow                        0                        0
34                 copp-system-p-class-important                 84547665                        0
35                  copp-system-p-class-critical                 11620514                        0
36                 copp-system-p-class-l3uc-data                        0                        0
38                                           FEX                        0                        0
39                           Internal DIAG(GOLD)                 30943792                        0
40                                        LC BFD               2833291020                        0
41                                        SUP TX                        0                        0
42                                     VXLAN OAM                        0                        0
43                                   Bellavue lb                        0                        0
44                                   SPAN-EGRESS                        0                        0
45                                         DOT1X                        0                        0
46                                      MPLS OAM                        0                        0
47                                       NETFLOW                        0                        0
48                              SSX/EHM MAC MOVE                        0                        0
51                                      UCS MGMT                        0                        0
52                                       FC_FCOE                        0                        0
53                                          MDNS                        0                        0
63                                 class-default                        0                        0
------------------------------------------------------------------------ 
```

The above output indicates that our CoPP class of interest, `copp-system-p-class-ndp`, has a queue number of 28.

### Filter Ethanalyzer on CoPP Class

Lastly, execute the `ethanalyzer local interface inband decode-internal display-filter "cisco.blob.sup_qnum==<xyz>" limit-captured-frames 0` command, replacing `<xyz>` with the queue number identified in the previous step (in our example, it's 28).

<blockquote>
<b>Note</b>: If you get an error message similar to either <pre>tshark: Neither "cisco.blob.sup_qnum" nor "35" are field or protocol names.</pre> or <pre>ethanalyzer: Neither "cisco.blob.sup_qnum" nor "28" are field or protocol names</pre>, you may be running into one of the following issues:
<ul>
<li>You most likely forgot to include the <pre>decode-internal</pre> keyword in the command.</li>
<li>You may be running an older NX-OS software release - you'll need to upgrade the NX-OS software of your switch to NX-OS 10.1(1) or later to filter on traffic from a specific CoPP class in Ethanalyzer.</li>
<li>You may be running a Nexus switch or line card that does not support this feature, such as a first-generation Nexus 9000 line card or switch (which use the Broadcom Trident 2 ASIC, model numbers typically end in TX or PX) or a Nexus 9800 series switch (which use the Cisco Silicon One ASIC). This feature is only supported on Nexus 9000 series switches with the Cisco Cloud Scale ASIC, which typically includes model numbers that end in -EX, -FX, -FX2, -FX3, -GX, -GX2, and so on.</li>
</ul>
</blockquote>

```
switch# ethanalyzer local interface inband decode-internal display-filter "cisco.blob.sup_qnum==28" limit-captured-frames 0
Capturing on 'ps-inb' 
Frame 2093: 142 bytes on wire (1136 bits), 142 bytes captured (1136 bits) on interface ps-inb, id 0
    Section number: 1
    Interface id: 0 (ps-inb)
        Interface name: ps-inb
    Encapsulation type: Ethernet (1)
    Arrival Time: Feb 10, 2024 17:02:21.999873423 UTC
    [Time shift for this packet: 0.000000000 seconds]
    Epoch Time: 1707584541.999873423 seconds
    [Time delta from previous captured frame: 0.001834959 seconds]
    [Time delta from previous displayed frame: 0.000000000 seconds]
    [Time since reference or first frame: 17.056774035 seconds]
    Frame Number: 2093
    Frame Length: 142 bytes (1136 bits)
    Capture Length: 142 bytes (1136 bits)
    [Frame is marked: False]
    [Frame is ignored: False]
    [Protocols in frame: eth:cisco:eth:ethertype:vlan:ethertype:ipv6:icmpv6]
Cisco Protocol
    Direction: IN 
    Top-16
        Top-8: 0000000000000000
        Next-8: 0000000000000000
    EPC-Dot1q
        Destination: 000002020202
        Source: 000004040404
        TPID: 34953
        PCP CFI VID: 34953
    Blob
        First 4 bytes of Blob: 0
        SUP Code: 255
        SUP Queue Number: 28
    PTP
        Timestamp: 165948901460801
    iETH
        Start of Frame: 251 (0xfb)
        Hdr Type: 0 (0x0)
        Ext Hdr: 0 (0x0)
        Opcode: 0 (0x0)
        Source Index: 5 (0x5)
        Dest Index: 1471 (0x5bf)
        Source Chip: 1 (0x1)
        Source Port: 17 (0x11)
        Dest Chip: 0 (0x0)
        Dest Port: 0 (0x0)
        Outer BD: 0 (0x0)
        BD: 2501 (0x9c5)
        Traceroute: 0 (0x0)
        Don't Learn: 0 (0x0)
        SPAN: 0 (0x0)
        Alter if possible: 0 (0x0)
        IP TTL Bypass: 0 (0x0)
        Source is Tunnel: 0 (0x0)
        Dest is Tunnel: 0 (0x0)
        L2 Tunnel: 0 (0x0)
        SUP Tx: 0 (0x0)
        SUP Code: 0 (0x0)
        COS De: 14 (0xe)
        TClass: 0 (0x0)
        SRC is peer: 0 (0x0)
        Packet Hash: 253 (0xfd)
Ethernet II, Src: 00:00:11:11:01:06, Dst: 20:20:00:00:00:aa
    Destination: 20:20:00:00:00:aa
        Address: 20:20:00:00:00:aa
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Source: 00:00:11:11:01:06
        Address: 00:00:11:11:01:06
        .... ..0. .... .... .... .... = LG bit: Globally unique address (factory default)
        .... ...0 .... .... .... .... = IG bit: Individual address (unicast)
    Type: 802.1Q Virtual LAN (0x8100)
802.1Q Virtual LAN, PRI: 7, DEI: 0, ID: 2501
    1.   .... .... .... = Priority: Network Control (7)
    ...0 .... .... .... = DEI: Ineligible
    .... 1001 1100 0101 = ID: 2501
    Type: IPv6 (0x86dd)
Internet Protocol Version 6, Src: 2001:db8:2:1::232, Dst: 2001:db8:2:1::1
    0110 .... = Version: 6
    .... 1110 0000 .... .... .... .... .... = Traffic Class: 0xe0 (DSCP: CS7, ECN: Not-ECT)
        .... 1110 00.. .... .... .... .... .... = Differentiated Services Codepoint: Class Selector 7 (56)
        .... .... ..00 .... .... .... .... .... = Explicit Congestion Notification: Not ECN-Capable Transport (0)
    .... 0000 0000 0000 0000 0000 = Flow Label: 0x00000
    Payload Length: 24
    Next Header: ICMPv6 (58)
    Hop Limit: 255
    Source Address: 2001:db8:2:1::232
    Destination Address: 2001:db8:2:1::1
Internet Control Message Protocol v6
    Type: Neighbor Advertisement (136)
    Code: 0
    Checksum: 0x2a13 [correct]
    [Checksum Status: Good]
    Flags: 0xc0000000, Router, Solicited
        1... .... .... .... .... .... .... .... = Router: Set
        .1.. .... .... .... .... .... .... .... = Solicited: Set
        ..0. .... .... .... .... .... .... .... = Override: Not set
        ...0 0000 0000 0000 0000 0000 0000 0000 = Reserved: 0
    Target Address: 2001:db8:2:1::232 
```

A detailed view of the traffic matching against this CoPP class will be displayed. This output can be quite lengthy in a production environment, but it can be used to identify key details (source/destination IP addressing, Layer 4 port information, etc.) about traffic that may be flooding this CoPP class and causing the incrementing drops.

## Use Cases

A few use cases where filtering Ethanalyzer on a specific CoPP class can be incredibly useful include:

* Identifying IP traffic that may be causing ARP glean drops in the `copp-system-p-class-l3uc-data` class.
* Identifying multicast traffic encountering RPF (Reverse Path Forwarding) failure in the `copp-system-p-class-l3mc-data` or `copp-system-p-class-undesirable` classes.
* Identifying IP traffic punted to the control plane due to TTL expiration or MTU issues in the `copp-system-p-class-exception-diag` class.
* Identifying IP traffic punted to the control plane due to the presence of IP options or (more commonly) a destination IP address that is unreachable due to a lack of routes in the routing table through the `copp-system-p-class-exception` class.

In a busy control plane, the above use cases can be quite challenging to troubleshoot without the ability to filter Ethanalyzer on a specific CoPP class. This feature can be greatly aid network operators troubleshooting these types of control plane issues on Cisco Nexus switches.

## Conclusion

Being able to identify *what* is causing an issue is paramount to resolving it. The ability to filter Ethanalyzer on a specific CoPP class enables network operators to identify what traffic is causing incrementing drops in a CoPP class, which can be a game-changer in troubleshooting control plane issues on Cisco Nexus switches or general network reachability issues.
