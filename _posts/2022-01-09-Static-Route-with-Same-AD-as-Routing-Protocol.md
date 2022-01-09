---
layout: post
title: Static Route with Same Administrative Distance as Dynamic Routing Protocol
---

## Overview

In the [Discord server](https://discord.com/invite/4Y9g9yzyBq) operated by [sysengineer](https://beacons.ai/sysengineer), the community has a Discord bot maintained by [Terranova Tech](https://www.youtube.com/channel/UC_bfWv6YbWJOdvnu_pbmvNw) that quizzes people studying for their CCNA and CCNP certifications. The bot asked the following CCNA-level question:

```
Which parameter can be tuned to affect the selection of a static route as a backup, when a dynamic protocol is also being used?

A. hop count
B. link bandwidth
C. link delay
D. link cost
```

The bot reported that the correct answer choice is `D. link cost`. A user in the Discord server pointed out that the correct answer is not correct because the question appears to be describing a floating static route. By definition, a floating static route for a prefix has a higher Administrative Distance than a dynamic routing protocol with the same prefix installed in the unicast routing table. The cost of a link may affect the dynamic routing protocol's metric for the path to the prefix, but it will not affect the Administrative Distance.

I theorized that this question may be referencing a scenario where the static route and the dynamic routing protocol share the same Administrative Distance. Past blog posts from [Jeremy Stretch](https://packetlife.net/blog/2008/dec/6/two-routing-protocols-same-ad/) and [Łukasz Szyda](http://netoperation.blogspot.com/2011/03/administrative-distance.html) tested this concept with two dynamic routing protocol's with the same Administrative Distance with varying results. On some Cisco IOS versions, the most recent routing protocol to offer a path would be inserted in the routing table. On other Cisco IOS versions, it seemed like the default Administrative Distance of the protocol would act as the tie-breaker and determine which protocol's path was inserted in the routing table. However, I have not yet found any discussions comparing a static route with the same Administrative Distance as a dynamic routing protocol.

## Topology

First, let's review our topology:

```
+------------+                    +------------+
|     R1     |Gi2              Gi2|     R2     |
| 100.1.1.1  +--------------------+ 100.2.2.2  |
|            |1.1.1.1      1.1.1.2|            |
+------------+     1.1.1.0/30     +------------+
```

We have two Cisco virtual routers, R1 and R2. Both are connected to each other through Layer 3 interface GigabitEthernet2 in the 1.1.1.0/30 subnet. R1's GigabitEthernet2 interface owns 1.1.1.1, while R2's GigabitEthernet2 interface owns 1.1.1.2. R1 and R2 have a Loopback0 interface. R1's Loopback0 interface owns 100.1.1.1/32, while R2's Loopback0 interface owns 100.2.2.2/32. Both routers are running OSPF; GigabitEthernet2 and Loopback0 of both routers are activated under the OSPF process, so both routers are fully OSPF adjacent with each other.

The testing described in this article were performed on the following nodes:

* Cisco CSR1000v routers running IOS-XE 17.03.04a
* Cisco Catalyst 8000v routers running IOS-XE 17.07.01a

The CLI output in this post is taken from the Catalyst 8000v routers running IOS-XE 17.07.01a, as they'll most likely have the most up-to-date behavior, CLI output, and debugs.

## Testing

First, let's take a look at the current path to 100.2.2.2/32 in R1's routing table.

```
R1#show ip route 100.2.2.2 255.255.255.255
Routing entry for 100.2.2.2/32
  Known via "ospf 1", distance 110, metric 2, type intra area
  Last update from 1.1.1.2 on GigabitEthernet2, 00:00:09 ago
  Routing Descriptor Blocks:
  * 1.1.1.2, from 100.2.2.2, 00:00:09 ago, via GigabitEthernet2
      Route metric is 2, traffic share count is 1
```

As you can see, 100.2.2.2/32 is learned from R2 via OSPF. The metric for this route is 2. Next, let's configure a static route for 100.2.2.2/32 with a next-hop IP address of 1.1.1.2 that has an Administrative Distance of 110.

```
R1#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
R1(config)#ip route 100.2.2.2 255.255.255.255 1.1.1.2 110
R1(config)#end
R1#
```

Next, let's see if the routing table's path to 100.2.2.2/32 has changed.

```
R1#show ip route 100.2.2.2 255.255.255.255
Routing entry for 100.2.2.2/32
  Known via "static", distance 110, metric 0
  Routing Descriptor Blocks:
  * 1.1.1.2
      Route metric is 0, traffic share count is 1
```

We can see that the static route overrode the path offered by OSPF. However, it's not clear if it's because:

1. The static route was configured *after* OSPF offered its path to the unicast routing table.
2. The static route's default Administrative Distance of 1 is superior to OSPF's Administrative Distance of 110.
3. The static route's metric (0) is superior to OSPF's metric (2).

We cannot test the second or third possibilities, as you cannot change the *default* Administrative Distance of either static routes or OSPF, and you cannot modify the metric of a static route the same way you can modify the metric of an OSPF path through link cost. However, we can test the first possibility by withdrawing the 100.2.2.2/32 route from the topology by removing the `network 100.2.2.2 0.0.0.0 area 0` configuration from R2, then re-adding it.

```
R2#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
R2(config)#router ospf 1
R2(config-router)#no network 100.2.2.2 0.0.0.0 area 0
R2(config-router)#network 100.2.2.2 0.0.0.0 area 0
R2(config-router)#end
R2#
```

If we check R1's routing table now, we can see that the static route is still present in R1's routing table. This means we've debunked the first possibility (where the routing table has a "recency bias" and installs the most recent path received for a prefix when Administrative Distance is tied between protocols), at least in modern Cisco IOS-XE software.

We can confirm this further by activating some debugs on R1.

```
R1#terminal monitor
R1#debug ip routing
IP routing debugging is on
R1#debug ip routing detail
IP routing debugging is on (detailed)
R1#debug ip routing static db
IP static routing local table debugging is on
R1#debug ip routing static detail
IP static routing detail debugging is on
R1#debug ip routing static event
IP static routing event debugging is on
R1#debug ip routing static route 100.2.2.2 255.255.255.255
IP static routing destination debugging is on, for 100.2.2.2 255.255.255.255
R1#debug ip ospf rib
OSPF RIB (Routing Information Base) debugging is on
OSPF Local RIB (Routing Information Base) debugging is on
OSPF Global RIB (Routing Information Base) debugging is on
OSPF Redistribution debugging is on
R1#
```

The current path in R1's routing table is the static route with an Administrative Distance of 110.

```
R1#show ip route 100.2.2.2 255.255.255.255
Routing entry for 100.2.2.2/32
  Known via "static", distance 110, metric 0
  Routing Descriptor Blocks:
  * 1.1.1.2
      Route metric is 0, traffic share count is 1
```

On R1, let's stop advertising 100.2.2.2/32 into OSPF.

```
R2#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
R2(config)#router ospf 1
R2(config-router)#no network 100.2.2.2 0.0.0.0 area 0
R2(config-router)#
```

OSPF debugs on R1 confirm that OSPF has reconverged and purged 100.2.2.2/32 from the OSPF LSDB (Link State Database).

```
*Jan  9 2022 19:22:29.061: OSPF-1 LRIB : Updating route 1.1.1.0/30
*Jan  9 2022 19:22:29.062: OSPF-1 LRIB :  Add path area 0, type Intra, dist 1,  forward 0, tag 0x0, via 1.1.1.1 GigabitEthernet2, route flags (Connected), path flags (Connected), source 100.2.2.2, spf 8, list-type change_list, src rtr 100.2.2.2
*Jan  9 2022 19:22:29.062: OSPF-1 LRIB : Updating route 100.1.1.1/32
*Jan  9 2022 19:22:29.062: OSPF-1 LRIB :  Add path area 0, type Intra, dist 1,  forward 0, tag 0x0, via 100.1.1.1 Loopback0, route flags (Connected), path flags (Connected), source 100.1.1.1, spf 8, list-type change_list, src rtr 100.1.1.1
*Jan  9 2022 19:22:29.063: OSPF-1 LRIB : Sync'ed 100.1.1.1/32 type Intra - change (0x0): added 0 paths, deleted 0 paths, spf 8, route instance 8, pdb spf instance 8
*Jan  9 2022 19:22:29.063: OSPF-1 LRIB : Sync'ed 1.1.1.0/30 type Intra - change (0x0): added 0 paths, deleted 0 paths, spf 8, route instance 8, pdb spf instance 8
*Jan  9 2022 19:22:29.063: OSPF-1 LRIB : Sync'ed 100.2.2.2/32 type Intra - change (RtDelete, RthDelete): added 0 paths, deleted 0 paths, spf 8, route instance 7, pdb spf instance 8
*Jan  9 2022 19:22:29.064: OSPF-1 LRIB : Purging first-hop via 1.1.1.2 on GigabitEthernet2
```

Next, let's re-advertise 100.2.2.2/32 into OSPF on R2.

```
R2(config-router)#network 100.2.2.2 0.0.0.0 area 0
R2(config-router)#
```

OSPF debugs on R1 confirm that OSPF reconverges, with 100.2.2.2/32 being re-added to the OSPF LSDB.

```
*Jan  9 2022 19:23:41.862: OSPF-1 LRIB : Updating route 1.1.1.0/30
*Jan  9 2022 19:23:41.862: OSPF-1 LRIB :  Add path area 0, type Intra, dist 1,  forward 0, tag 0x0, via 1.1.1.1 GigabitEthernet2, route flags (Connected), path flags (Connected), source 100.2.2.2, spf 9, list-type change_list, src rtr 100.2.2.2
*Jan  9 2022 19:23:41.863: OSPF-1 LRIB : Updating route 100.1.1.1/32
*Jan  9 2022 19:23:41.863: OSPF-1 LRIB :  Add path area 0, type Intra, dist 1,  forward 0, tag 0x0, via 100.1.1.1 Loopback0, route flags (Connected), path flags (Connected), source 100.1.1.1, spf 9, list-type change_list, src rtr 100.1.1.1
*Jan  9 2022 19:23:41.863: OSPF-1 LRIB : Creating new first-hop via 1.1.1.2 on GigabitEthernet2
*Jan  9 2022 19:23:41.864: OSPF-1 LRIB : Creating route 100.2.2.2/32
*Jan  9 2022 19:23:41.864: OSPF-1 LRIB :  Add path area 0, type Intra, dist 2,  forward 0, tag 0x0, via 1.1.1.2 GigabitEthernet2, route flags (None), path flags (none), source 100.2.2.2, spf 9, list-type change_list, src rtr 100.2.2.2
```

R1's OSPF debugs show that OSPF offers this new path to the routing table.

```
*Jan  9 2022 19:23:41.865: RT: updating ospf 100.2.2.2/32 (0x0) [local lbl/ctx:1048577/0x0] omp-tag:0  :
    via 1.1.1.2 Gi2  0 0 0x0 1048578 0x100001
```

However, the routing table of R1 rejects the OSPF path.

```
*Jan  9 2022 19:23:41.865: RT: rib update return code: 17
*Jan  9 2022 19:23:41.865: OSPF-1 GRIB :   IP route replace of 1 next hops failed for 100.2.2.2/32 (flags 0x0, type Intra, tag 0x0), retcode 17
*Jan  9 2022 19:23:41.865: OSPF-1 GRIB :   Next hop via 1.1.1.2 on GigabitEthernet2 (distance 2, source 100.2.2.2, label 1048578) NOT installed
```

Unfortunately, the routing table does not show us *why* this path was rejected. However, the fact that the path was rejected shows that the routing table does not factor in the recency of a path when attempting to tie-break multiple paths with the same Administrative Distance.

Next, let's see what R1's debugs show when the static route for 100.2.2.2/32 is removed, then re-configured. First, let's remove the static route for 100.2.2.2/32.

```
R1#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
R1(config)#no ip route 100.2.2.2 255.255.255.255 1.1.1.2 110
R1(config)#
```

R1's debugs show that the static route is removed, and the OSPF path successfully replaces it.

```
*Jan  9 2022 19:25:10.144: IP-ST-DB(default):  static_no_free_sre() sre: 0x7F49170F9A20
*Jan  9 2022 19:25:10.145:  100.2.2.2/32 via 1.1.1.2  sr_pol:  bind 0x100001 ,tag 0,fg 0x40020004,dis 110,name ,lfg 0x402,own M
*Jan  9 2022 19:25:10.145: RT: del 100.2.2.2 via 1.1.1.2, static metric [110/0]
*Jan  9 2022 19:25:10.146: RT: delete subnet route to 100.2.2.2/32
*Jan  9 2022 19:25:10.147: OSPF-1 GRIB : Add backup request 100.2.2.2/32, type Intra
*Jan  9 2022 19:25:10.147: IP-ST-DB(default):  ip_free_sre() Removed sre->name freeing before sr policy delete
*Jan  9 2022 19:25:10.148: IP-ST-DB(default):  ip_free_sre() sre: 0x7F49170F9A20, ref: 1
*Jan  9 2022 19:25:10.167: RT: updating ospf 100.2.2.2/32 (0x0) [local lbl/ctx:1048577/0x0] omp-tag:0  :
    via 1.1.1.2 Gi2  0 0 0x0 1048578 0x100001

*Jan  9 2022 19:25:10.169: RT: add 100.2.2.2/32 via 1.1.1.2, ospf metric [110/2]
*Jan  9 2022 19:25:10.170: OSPF-1 GRIB :   IP route replace of 1 next hops succeeded for 100.2.2.2/32 (flags 0x0, type Intra, tag 0x0), retcode 0
*Jan  9 2022 19:25:10.170: OSPF-1 GRIB :   Next hop via 1.1.1.2 on GigabitEthernet2 (distance 2, source 100.2.2.2, label 1048578) installed
*Jan  9 2022 19:25:10.170: OSPF-1 LRIB : Sync'ed 100.2.2.2/32 type Intra - change (Change, PathChange, ForcedSync): added 1 paths, deleted 0 paths, spf 9, route instance 9, pdb spf instance 9
```

We can confirm this by looking at the state of the routing table for the 100.2.2.2/32 prefix.

```
R1(config)#do show ip route 100.2.2.2 255.255.255.255
Routing entry for 100.2.2.2/32
  Known via "ospf 1", distance 110, metric 2, type intra area
  Last update from 1.1.1.2 on GigabitEthernet2, 00:01:24 ago
  Routing Descriptor Blocks:
  * 1.1.1.2, from 100.2.2.2, 00:01:24 ago, via GigabitEthernet2
      Route metric is 2, traffic share count is 1
R1(config)#
```

Next, let's re-configure the static route for 100.2.2.2/32 with an Administrative Distance of 110.

```
R1(config)#ip route 100.2.2.2 255.255.255.255 1.1.1.2 110
R1(config)#
```

R1's debugs show that the static route successfully replaces the OSPF route in the routing table.

```
*Jan  9 2022 19:26:48.686: IP-ST(default): updating same distance on 100.2.2.2/32
*Jan  9 2022 19:26:48.686: IP-ST(default):  100.2.2.2/32 [110], 1.1.1.2 Path = 8, no change, not active state
*Jan  9 2022 19:26:48.688: IP-ST-DB(default): ip_addstatic_route(), succeed
*Jan  9 2022 19:26:48.688:  100.2.2.2/32 via 1.1.1.2 ,tag 0,fg 0x40020004,dis 110,name ,lfg 0x0,own M sr_policy:
*Jan  9 2022 19:26:48.690: IP-ST(default):  100.2.2.2/32 [110], 1.1.1.2 Path = 1 2 3 7
*Jan  9 2022 19:26:48.692: RT: updating static 100.2.2.2/32 (0x0) [local lbl/ctx:1048577/0x0] omp-tag:0  :
    via 1.1.1.2   0 0 0x0 1048578 0x100001

*Jan  9 2022 19:26:48.692: RT: closer admin distance for 100.2.2.2, flushing 1 routes
*Jan  9 2022 19:26:48.693: OSPF-1 GRIB : Route 100.2.2.2/32 type 'Intra' has been replaced
*Jan  9 2022 19:26:48.693: RT: add 100.2.2.2/32 via 1.1.1.2, static metric [110/0], add succeed, active state
```

In these debugs, note the `RT: closer admin distance for 100.2.2.2, flushing 1 routes` line that happened at 19:26:48.692 router time. This is a debug from the routing table suggesting that a superior Administrative Distance for 100.2.2.2 is causing a previous path to be purged from the routing table. This raises an important question - is the "closer admin distance" cited by this debug the *current* Administrative Distance of each protocol, or the *default* Administrative Distance of each protocol?

The previous set of debugs shows what happens when a static route has the same Administrative Distance as a dynamic routing protocol. This is an admittedly unusual situation. A common tactic I like to use when troubleshooting an issue or trying to reverse-engineer the behavior of a system is to compare a "broken state" (the state I'm actively troubleshooting) with a "working state" (a state that I know is not broken and exhibits logical behavior). If we consider the previous set of debugs our "broken state", a good test to determine whether these debugs are citing the *current* Administrative Distance or the *default* Administrative Distance would be to compare the previous set of debugs with a set of debugs taken when the static route has a lower Administrative Distance than the routing protocol.

Let's test this scenario by removing the current static route, and configuring an identical static route with an Administrative Distance of 100.

```
R1(config)#no ip route 100.2.2.2 255.255.255.255 1.1.1.2 110
R1(config)#ip route 100.2.2.2 255.255.255.255 1.1.1.2 100
R1(config)#
```

The corresponding debugs from R1 are as follows:

```
*Jan  9 2022 19:28:52.557: IP-ST(default): updating same distance on 100.2.2.2/32
*Jan  9 2022 19:28:52.558: IP-ST(default):  100.2.2.2/32 [100], 1.1.1.2 Path = 8, no change, not active state
*Jan  9 2022 19:28:52.559: IP-ST-DB(default): ip_addstatic_route(), succeed
*Jan  9 2022 19:28:52.559:  100.2.2.2/32 via 1.1.1.2 ,tag 0,fg 0x40020004,dis 100,name ,lfg 0x0,own M sr_policy:
*Jan  9 2022 19:28:52.563: IP-ST(default):  100.2.2.2/32 [100], 1.1.1.2 Path = 1 2 3 7
*Jan  9 2022 19:28:52.565: RT: updating static 100.2.2.2/32 (0x0) [local lbl/ctx:1048577/0x0] omp-tag:0  :
    via 1.1.1.2   0 0 0x0 1048578 0x100001

*Jan  9 2022 19:28:52.566: RT: closer admin distance for 100.2.2.2, flushing 1 routes
*Jan  9 2022 19:28:52.566: OSPF-1 GRIB : Route 100.2.2.2/32 type 'Intra' has been replaced
*Jan  9 2022 19:28:52.566: RT: add 100.2.2.2/32 via 1.1.1.2, static metric [100/0], add succeed, active state
```

If we compare these debugs (our "working state") with the previous debugs (our "broken state"), we can see that they are basically identical aside from the Administrative Distance value (100 vs. 110) and the debug timestamps.

## Conclusion

Unfortunately, our testing was unable to conclusively reverse engineer the tie-breaker behavior when two protocols with the same Administrative Distance offer paths to the routing table for the same prefix. However, based upon the behavior we can observe, it seems that modern Cisco IOS-XE software utilizes the default Administrative Distance of each protocol as a tie-breaker, with the default Administrative Distance of a static route most likely being a value of 1.
