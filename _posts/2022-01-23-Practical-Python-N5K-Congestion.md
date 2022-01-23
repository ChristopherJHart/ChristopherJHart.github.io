---
layout: post
title: Practical Python - Identifying Congested Interfaces of Nexus 5500 Switches
---

## Overview

A common support case I have seen during my career in Cisco TAC is where customers observe intermittently incrementing input discard counters on one or more interfaces of a Cisco Nexus 5500 series switch. This symptom is usually followed by reports of connectivity issues, packet loss, or application latency for traffic flows traversing the switch.

```
switch# show interface
<snip>
Ethernet1/1 is up
    0 input with dribble  28957683 input discard
Ethernet1/2 is up
    0 input with dribble  0 input discard
Ethernet1/3 is up
    0 input with dribble  782912 input discard
Ethernet1/4 is up
    0 input with dribble  2837553 input discard
Ethernet1/5 is up
    0 input with dribble  0 input discard
```

Oftentimes, this issue is highly intermittent and unpredictable. Once every few hours (or sometimes even days), the input discards will increment over the course of a few dozen seconds, then stop. This is a network administrator's worst nightmare - an intermittent issue that is impossible to predict and only happens for a short period of time, but has major impact to the network.

In this post, we will briefly explore what input discards mean, the challenges with troubleshooting this intermittent issue manually, and how we can use Python to help troubleshoot.

## What are Input Discards?

Cisco Nexus 5000, 6000, and 7000 switches utilize a Virtual Output Queue (VOQ) queuing architecture for unicast traffic. This means that when a packet enters the switch, the switch does the following:

1. Parses the packet's headers.
2. Makes a forwarding decision on the packet, thus determining the packet's egress interface.
3. Buffers the packet in memory on the ingress interface within a virtual queue (VQ) dedicated to the egress interface until the egress interface can transmit the packet.

An interface becomes congested when the total sum of traffic that needs to be transmitted out of the interface exceeds the bandwidth of the interface itself. For example, let's say that Ethernet1/1, Ethernet1/2, and Ethernet1/3 of a switch are all interfaces with 10Gbps of bandwidth.

![]({{ site.baseurl }}/images/2022/practical-python-n5k-congestion/topology.png)

Let's also say that 7.5Gbps of traffic ingresses the switch through Ethernet1/1, and 7.5Gbps of traffic ingresses the switch through Ethernet1/2.

![]({{ site.baseurl }}/images/2022/practical-python-n5k-congestion/switch_ingress_traffic.png)

All of this traffic needs to egress Ethernet1/3. This means a combined sum of 15Gbps of traffic needs to egress Ethernet1/3. However, Ethernet1/3 is a 10Gbps interface; it's only capable of transmitting up to 10Gbps of traffic at a time. Therefore, the excess 5Gbps of traffic *must* be buffered by the switch. This traffic will be buffered at the ingress interfaces (Ethernet1/1 and Ethernet1/2) within a virtual queue that represents Ethernet1/3.

![]({{ site.baseurl }}/images/2022/practical-python-n5k-congestion/switch_virtual_queues.png)

> **Note**: In this picture, there is a total of three interfaces, so each interface only has two virtual queues. In reality, most switches have anywhere from 24 to 48 ports, sometimes more. Each potential egress interface gets its own unique virtual queue. Also, note that the ingress buffer for Ethernet1/3 is not pictured here since it is not relevant - in reality, Ethernet1/3 would have ingress buffers.

If Ethernet1/3 remains congested for a "long" period of time, then the virtual queue for Ethernet1/3 on the ingress buffer of Ethernet1/1 and Ethernet1/2 will become full, and the switch will not be able to accept new packets destined to Ethernet1/3. Additional packets are then dropped on ingress, and the "input discards" counter is incremented on the ingress interface (Ethernet1/1 and Ethernet1/2). It's important to note that no counter will be incremented on the congested egress interface.

The word "long" is purposefully ambiguous because it is subjective. Within a matter of milliseconds, an egress interface can become congested, ingress buffers can fill, and input discards can begin to increment on a switch's interfaces. A few milliseconds is a very short amount of time to a human being, but to a computer, a few milliseconds can be a lifetime.

The two key points of this section are:

1. Input discards in a VOQ queuing architecture indicate one or more congested egress interfaces
2. There is no counter on an interface that identifies it is congested

## Manually Troubleshooting Input Discards on Nexus 5500 Switches

Ultimately, network congestion is a system throughput problem. There are two fundamental ways to solve a throughput problem in a system:

1. Identify the bottleneck of the system and improve it.
2. Reduce the flow of the system until the bottleneck no longer exists.

Option #2 is not very helpful because network engineers often have little or no control over the amount of traffic within the network, which leaves us with Option #1. Dr. Goldratt's Theory of Constraints states that making improvements anywhere but the bottleneck of a system will not improve the throughput of the system. Therefore, identifying a congested egress interface is extremely important to solving network congestion.

Nexus 5500 series switches offer a command - `show hardware internal carmel asic <x> registers match .*STA.*frh.*` - that displays a set of ASIC registers that match the regular expression `.*STA.*frh.*`. Registers matching this pattern identify the amount of data stored in a specific egress interface's virtual queue at the time of the command's execution.

```
switch# show hardware internal carmel asic 0 registers match .*STA.*frh.*
<snip>
    Slot 0 Carmel 0 register contents:
    Register Name                                          | Offset   | Value
    -------------------------------------------------------+----------+-----------
    car_bm_STA_frh0_addr_0                                 | 0x5031c  | 0
    car_bm_STA_frh0_addr_1                                 | 0x5231c  | 0
    car_bm_STA_frh0_addr_2                                 | 0x5431c  | 0
    car_bm_STA_frh0_addr_3                                 | 0x5631c  | 0
    car_bm_STA_frh0_addr_4                                 | 0x5831c  | 0
    car_bm_STA_frh0_addr_5                                 | 0x5a31c  | 0x4
```

The output of this command shows that the interface corresponding with ASIC 0's memory address 5 (indicated by the `addr_5` substring of the register) contained data when the command was executed. This output does *not* show historical data - it's a snapshot of the ASIC's buffers at the time the command is executed.

We can translate the ASIC number and memory address to an egress interface we recognize using the `show hardware internal carmel all-ports` command.

```
switch# show hardware internal carmel all-ports 
<snip>
Carmel Port Info:
name   |log|car|mac|flag|adm|opr|m:s:l|ipt|fab|xcar|xpt|if_index|diag|ucVer
-------+---+---+---+----+---+---+-----+---+---+----+---+--------+----+-----
xgb1/2 |1  |0  |0 -|b7  |en |up |0:0:f|0  |92 |0   |0  |1a001000|pass| 4.0b
xgb1/1 |0  |0  |1 -|b7  |en |up |1:1:f|1  |88 |0   |0  |1a000000|pass| 4.0b
xgb1/4 |3  |0  |2 -|b7  |en |up |2:2:f|2  |93 |0   |0  |1a003000|pass| 4.0b
xgb1/3 |2  |0  |3 -|b7  |en |up |3:3:f|3  |89 |0   |0  |1a002000|pass| 4.0b
xgb1/6 |5  |0  |4 -|b7  |dis|dn |4:4:f|4  |90 |0   |0  |1a005000|pass| 4.0b
xgb1/5 |4  |0  |5 -|b7  |en |up |5:5:f|5  |94 |0   |0  |1a004000|pass| 4.0b
xgb1/8 |7  |0  |6 -|b7  |dis|dn |6:6:f|6  |95 |0   |0  |1a007000|pass| 4.0b
xgb1/7 |6  |0  |7 -|b7  |dis|dn |7:7:f|7  |91 |0   |0  |1a006000|pass| 4.0b
```

This table has three columns we should pay attention to - "name", "car", and "mac". The "car" column (which stands for "Carmel", the internal name of the Nexus 5500 ASIC) maps to the ASIC identifier. The "mac" column maps to the memory address. The "name" column contains the internal name of the switch's interface, the numbering of which (e.g. 1/1, 1/2, etc.) maps to the external-facing interface we're familiar with (e.g. Ethernet1/1, Ethernet1/2, etc.).

The ASIC registers displayed by the `show hardware internal carmel asic <x> registers match .*STA.*frh.*` command indicates that memory address 5 of ASIC 0 contained data when the command was executed. The table displayed by the `show hardware internal carmel all-ports` command translates this to internal interface xgb1/5, the numbering of which means that interface Ethernet1/5 contained data when the command was executed.

When network congestion is constantly occurring (meaning, input discards are constantly incrementing on one or more interfaces), you can execute these commands rapidly and get a very good idea as to which interface is your congested interface. However, these commands are not very useful when network congestion is intermittent and strikes over a small period of time. Most organizations cannot expect an employee to monitor interfaces for incrementing input discards over several hours *and* parse the collected data without error.

Thankfully, this is a perfectly reasonable expectation for a Python script!

## Automatically Troubleshooting Input Discards on Nexus 5500 Switches

[I created and published a Python script that solves this problem](https://github.com/ChristopherJHart/n5k-congested-interface-finder). It connects to a Nexus 5500 switch over SSH and rapidly (~10-15 checks per second) monitors a single interface for incrementing input discards.

```
$ python n5k_congested_interface_finder.py 192.0.2.10 Ethernet1/31 --username admin --password Password\!123
Connecting to device '192.0.2.10' with username 'admin'...
Successfully connected to device!
[2022-01-19 17:54:02.516177] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:02.614450] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:02.696916] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:02.774517] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:02.853212] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:02.933895] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:03.014426] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:03.095528] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:03.176304] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:03.256125] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:03.336812] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:03.416981] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:03.500254] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:03.595615] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:03.675079] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:03.754537] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:03.842111] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:03.927128] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:04.011785] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:04.096491] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:04.180880] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:04.306469] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:04.392104] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:04.504154] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:04.636330] Checking for input discards on interface Ethernet1/31...
```

The script identifies when input discards start to increment and collects ASIC register information:

```
[2022-01-19 17:54:12.358728] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:12.358728] 71915 new input discards found on Ethernet1/31, 1 ASIC registers!
[2022-01-19 17:54:14.702041] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:14.702041] 1748955 new input discards found on Ethernet1/31, 1 ASIC registers!
[2022-01-19 17:54:17.081937] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:17.081937] 2523546 new input discards found on Ethernet1/31, 1 ASIC registers!
[2022-01-19 17:54:20.453788] Checking for input discards on interface Ethernet1/31...
[2022-01-19 17:54:20.453788] 1749266 new input discards found on Ethernet1/31, 1 ASIC registers!
[2022-01-19 17:54:22.772208] Checking for input discards on interface Ethernet1/31...
```

After the script is interrupted with Control+C, ASIC register data is parsed and a report is generated summarizing the registers found each time incrementing input discards was detected. ASIC registers are conveniently translated to their corresponding external-facing names that network administrators are familiar with:

```
[2022-01-19 17:54:22.772208] Checking for input discards on interface Ethernet1/31...
^C
Raw results:

2022-01-19 17:54:12.358728: 2212147758 input discards (+71915)
    Ethernet1/10 (S0A1 0): 206
2022-01-19 17:54:14.702041: 2213896713 input discards (+1748955)
    Ethernet1/10 (S0A1 0): 197
2022-01-19 17:54:17.081937: 2216420259 input discards (+2523546)
    Ethernet1/10 (S0A1 0): 202
2022-01-19 17:54:20.453788: 2218169525 input discards (+1749266)
    Ethernet1/10 (S0A1 0): 198

Top Talkers:
Ethernet1/10 (S0A1 0): 803
```

Overall, this script took about 4-6 hours to develop, test, and refine. In return, I hope it will save engineers countless hours tracking down congested egress interfaces and improve the reliability of the world's networks!
