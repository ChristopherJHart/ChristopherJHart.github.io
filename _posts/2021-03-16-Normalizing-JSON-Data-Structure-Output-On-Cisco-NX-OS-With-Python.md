---
layout: post
title: Normalizing JSON Data Structure Output on Cisco NX-OS with Python
---

Earlier this year, [Ivan Pepelnjak wrote a blog post](https://blog.ipspace.net/2021/01/xml-json-cisco-nexus.html) on how the way network operating systems (including Cisco's NX-OS) convert XML-based data structures to JSON can cause certain keys to be either a dictionary or a list, depending on the quantity of elements nested within a parent element. Ivan's blog post does an excellent job of providing multiple workarounds for this issue, such as using an Ansible filter or using NX-API with the cli_array method to normalize the data. However, neither of these workarounds are applicable to you if you have either of the following use cases:

1. You need to parse structured output from a Nexus switch received via SSH (such as with the Netmiko or Scrapli libraries).
2. You need to parse strucutred output from a Nexus switch using a on-the-box Python script.

The purpose of this post is twofold:

1. Demonstrate this problem where JSON output from NX-OS may contain either a dictionary or a list, depending on the quantity of elements nested within a parent element.
2. Provide a Python 3 utility function that can normalize this data for the aforementioned use cases instead of leveraging NX-API or Ansible.

## The Problem

Ivan's blog post does an excellent job of demonstrating this problem, but to recap, I'll demonstrate my own example of the issue. This example was recreated using Nexus 9300v switches in CML2.1.

Consider a very simple topology where two Nexus 9000 switches (hostnames N9K-1 and N9K-2) are connected to each other via routed interface Ethernet1/1. An EIGRP process in ASN 1 is activated on Ethernet1/1.

```
N9K-1# show ip interface brief

IP Interface Status for VRF "default"(1)
Interface            IP Address      Interface Status
Lo0                  100.1.1.1       protocol-up/link-up/admin-up
Eth1/1               10.1.0.0        protocol-up/link-up/admin-up

N9K-1# show cdp neighbors interface Ethernet1/1
Capability Codes: R - Router, T - Trans-Bridge, B - Source-Route-Bridge
                  S - Switch, H - Host, I - IGMP, r - Repeater,
                  V - VoIP-Phone, D - Remotely-Managed-Device,
                  s - Supports-STP-Dispute

Device-ID          Local Intrfce  Hldtme Capability  Platform      Port ID
N9K-2(9JXT8T8H5KL)
                    Eth1/1         133    R S s     N9K-C9300v    Eth1/1

Total entries displayed: 1

N9K-2# show ip interface brief

IP Interface Status for VRF "default"(1)
Interface            IP Address      Interface Status
Lo0                  100.2.2.2       protocol-up/link-up/admin-up
Eth1/1               10.1.0.1        protocol-up/link-up/admin-up
N9K-2# show cdp neighbors interface Ethernet1/1
Capability Codes: R - Router, T - Trans-Bridge, B - Source-Route-Bridge
                  S - Switch, H - Host, I - IGMP, r - Repeater,
                  V - VoIP-Phone, D - Remotely-Managed-Device,
                  s - Supports-STP-Dispute

Device-ID          Local Intrfce  Hldtme Capability  Platform      Port ID
N9K-1(9EKQZIXEMGZ)
                    Eth1/1         149    R S s     N9K-C9300v    Eth1/1

Total entries displayed: 1

N9K-1# show running-config eigrp
<snip>
feature eigrp

router eigrp 1

interface loopback0
  ip router eigrp 1

interface Ethernet1/1
  ip router eigrp 1

N9K-2# show running-config eigrp
<snip>
feature eigrp

router eigrp 1

interface loopback0
  ip router eigrp 1

interface Ethernet1/1
  ip router eigrp 1
```

As expected, an EIGRP adjacency has formed between the two switches through Ethernet1/1.

```
N9K-1# show ip eigrp neighbors
IP-EIGRP neighbors for process 1 VRF default
H   Address                 Interface       Hold  Uptime  SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
0   10.1.0.1                Eth1/1          12   00:45:45  6    50    0   4

N9K-2# show ip eigrp neighbors
IP-EIGRP neighbors for process 1 VRF default
H   Address                 Interface       Hold  Uptime  SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
0   10.1.0.0                Eth1/1          10   00:45:42  16   96    0   4
```

If we analyze the output of the `show ip eigrp neighbors` command in a JSON data structure with the, we can see that all keys with the phrase "ROW_" in it (such as `ROW_asn`, `ROW_vrf`, and `ROW_peer`) have a corresponding value that is a dictionary.

```json
{
    "TABLE_asn": {
        "ROW_asn": {
            "asn": "1",
            "TABLE_vrf": {
                "ROW_vrf": {
                    "vrf": "default",
                    "TABLE_peer": {
                        "ROW_peer": {
                            "peer_handle": "0",
                            "peer_ipaddr": "10.1.0.1",
                            "peer_ifname": "Eth1/1",
                            "peer_holdtime": "13",
                            "peer_srtt": "6",
                            "peer_rto": "50",
                            "peer_xmitq_count": "0",
                            "peer_last_seqno": "4",
                            "peer_uptime": "PT46M49S"
                        }
                    }
                }
            }
        }
    }
}
```

Next, let's create a new EIGRP process in ASN 2 on both switches and activate this new EIGRP process on Ethernet1/1.

```
N9K-1# configure terminal
Enter configuration commands, one per line. End with CNTL/Z.
N9K-1(config)# router eigrp 2
N9K-1(config-router)# interface Ethernet1/1
N9K-1(config-if)# ip router eigrp 2
N9K-1(config-if)# end
N9K-1#

N9K-2# configure terminal
Enter configuration commands, one per line. End with CNTL/Z.
N9K-2(config)# router eigrp 2
N9K-2(config-router)# interface Ethernet1/1
N9K-2(config-if)# ip router eigrp 2
N9K-2(config-if)# end
N9K-2#
```

We can see that there are two EIGRP adjacencies formed over Ethernet1/1 - one in EIGRP process 1, the other in EIGRP process 2.

```
N9K-1# show ip eigrp neighbors
IP-EIGRP neighbors for process 1 VRF default
H   Address                 Interface       Hold  Uptime  SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
0   10.1.0.1                Eth1/1          11   00:49:03  6    50    0   4
IP-EIGRP neighbors for process 2 VRF default
H   Address                 Interface       Hold  Uptime  SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
0   10.1.0.1                Eth1/1          13   00:00:13  7    50    0   3

N9K-2# show ip eigrp neighbors
IP-EIGRP neighbors for process 1 VRF default
H   Address                 Interface       Hold  Uptime  SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
0   10.1.0.0                Eth1/1          12   00:49:06  16   96    0   4
IP-EIGRP neighbors for process 2 VRF default
H   Address                 Interface       Hold  Uptime  SRTT   RTO  Q  Seq
                                            (sec)         (ms)       Cnt Num
0   10.1.0.0                Eth1/1          14   00:00:16  1283 5000  0   3
```

Let's take another look at the JSON output of the `show ip eigrp neighbors` command.

```json
{
    "TABLE_asn": {
        "ROW_asn": [
            {
                "asn": "1",
                "TABLE_vrf": {
                    "ROW_vrf": {
                        "vrf": "default",
                        "TABLE_peer": {
                            "ROW_peer": {
                                "peer_handle": "0",
                                "peer_ipaddr": "10.1.0.1",
                                "peer_ifname": "Eth1/1",
                                "peer_holdtime": "12",
                                "peer_srtt": "6",
                                "peer_rto": "50",
                                "peer_xmitq_count": "0",
                                "peer_last_seqno": "4",
                                "peer_uptime": "PT1H9M33S"
                            }
                        }
                    }
                }
            },
            {
                "asn": "2",
                "TABLE_vrf": {
                    "ROW_vrf": {
                        "vrf": "default",
                        "TABLE_peer": {
                            "ROW_peer": {
                                "peer_handle": "0",
                                "peer_ipaddr": "10.1.0.1",
                                "peer_ifname": "Eth1/1",
                                "peer_holdtime": "10",
                                "peer_srtt": "7",
                                "peer_rto": "50",
                                "peer_xmitq_count": "0",
                                "peer_last_seqno": "3",
                                "peer_uptime": "PT20M43S"
                            }
                        }
                    }
                }
            }
        ]
    }
}
```

We can see that the value of the `ROW_asn` key is now a **list**, not a **dictionary**. This is problematic from a Python perspective because the way that one consumes the data in a dictionary is different from the way that one consumes the data in a list. It's not impossible to write code that can handle scenarios where the value of a dictionary key is either a dictionary or a list, but doing so can be difficult to consume, maintain, and test.

It would be *much* easier for us if every key containing the phrase "ROW_" was a list - whether it contains one element, or multiple elements. If you are running NX-OS software release 9.3(1) or later, you can use the `json native` pipe instead of the `json` pipe to accomplish this, as shown below.

```
N9K-1# show ip eigrp neighbors | json-pretty native
{
    "TABLE_asn": {
        "ROW_asn": [
            {
                "asn": 1,
                "TABLE_vrf": {
                    "ROW_vrf": [
                        {
                            "vrf": "default",
                            "TABLE_peer": {
                                "ROW_peer": [
                                    {
                                        "peer_handle": 0,
                                        "peer_ipaddr": "10.1.0.1",
                                        "peer_ifname": "Eth1/1",
                                        "peer_holdtime": 13,
                                        "peer_srtt": 6,
                                        "peer_rto": 50,
                                        "peer_xmitq_count": 0,
                                        "peer_last_seqno": 4,
                                        "peer_uptime": "PT16H7M38S"
                                    }
                                ]
                            }
                        }
                    ]
                }
            },
            {
                "asn": 2,
                "TABLE_vrf": {
                    "ROW_vrf": [
                        {
                            "vrf": "default",
                            "TABLE_peer": {
                                "ROW_peer": [
                                    {
                                        "peer_handle": 0,
                                        "peer_ipaddr": "10.1.0.1",
                                        "peer_ifname": "Eth1/1",
                                        "peer_holdtime": 13,
                                        "peer_srtt": 7,
                                        "peer_rto": 50,
                                        "peer_xmitq_count": 0,
                                        "peer_last_seqno": 3,
                                        "peer_uptime": "PT15H18M47S"
                                    }
                                ]
                            }
                        }
                    ]
                }
            }
        ]
    }
}
```

However, at the time of this writing, NX-OS 9.3(1) and later are the upcoming long-lived release, not the current recommended NX-OS software release. If you are not yet running NX-OS software where the `json native` pipe is available, you'll need an alternative solution.

## The Solution

The `normalize_output` utility function below modifies a dictionary `input` (which will be your JSON data structure obtained through NX-OS Python libraries) such that all dictionary keys containing the phrase "ROW_" are converted to lists of dictionaries. The modified dictionary `input` is then returned for further analysis.

```python
def normalize_output(input: dict) -> dict:
    for k, v in input.items():
        if "ROW_" in k and isinstance(v, dict):
            input[k] = [normalize_output(v)]
        elif isinstance(v, dict) and any("ROW_" in x for x in v.keys()):
            input[k] = normalize_output(v)
        # Taste to see if dictionary value is a list and if the list
        # contains dictionaries. This prevents us from needlessly normalizing
        # leaf nodes in the data structure.
        elif isinstance(v, list) and isinstance(v[0], dict):
            for index, item in enumerate(v):
                input[k][index] = normalize_output(item)
    return input
```

This utility function and its corresponding unit tests can be found in the [Normalize NX-OS JSON Data Structures GitHub repository](https://github.com/ChristopherJHart/normalize-nxos-json-data-structures).

If we run the structured output of the `show ip eigrp neighbors` command from a switch with multiple EIGRP processes configured through this utility function, we'll have the data structure shown below.

```json
{
    "TABLE_asn": {
        "ROW_asn": [
            {
                "asn": "1",
                "TABLE_vrf": {
                    "ROW_vrf": [
                        {
                            "vrf": "default",
                            "TABLE_peer": {
                                "ROW_peer": [
                                    {
                                        "peer_handle": "0",
                                        "peer_ipaddr": "10.1.0.1",
                                        "peer_ifname": "Eth1/1",
                                        "peer_holdtime": "12",
                                        "peer_srtt": "6",
                                        "peer_rto": "50",
                                        "peer_xmitq_count": "0",
                                        "peer_last_seqno": "4",
                                        "peer_uptime": "PT1H9M33S"
                                    }   
                                ]
                            }
                        }
                    ]
                }
            },
            {
                "asn": "2",
                "TABLE_vrf": {
                    "ROW_vrf": [
                        {
                            "vrf": "default",
                            "TABLE_peer": {
                                "ROW_peer": [
                                    {
                                        "peer_handle": "0",
                                        "peer_ipaddr": "10.1.0.1",
                                        "peer_ifname": "Eth1/1",
                                        "peer_holdtime": "10",
                                        "peer_srtt": "7",
                                        "peer_rto": "50",
                                        "peer_xmitq_count": "0",
                                        "peer_last_seqno": "3",
                                        "peer_uptime": "PT20M43S"
                                    }
                                ]
                            }
                        }
                    ]
                }
            }
        ]
    }
}
```

Although this data structure is larger, traversing through this data structure in Python is significantly easier. We can safely assume that all dictionary keys with the phrase "ROW_" have a value that is a list. This let's us iterate through the data using a for loop instead of tasting the dictionary's value and using `isinstance()` to correctly handle the data.

## Examples

Showcasing the purpose and use case of this utility function is best done through examples. The below three examples involve a scenario where you need to build some automation that reports the number of EIGRP adjacencies across your entire fleet of Nexus 9000 switches. Each example works within different business requirements that mold the implementation of the automation solution.

### Example One - JSON Output From Netmiko

The business requirements provided to you for this automation restricts you from gathering this information via NX-API or SNMP, but allows access to the switch via SSH.

An example of the Python script you might build for this task can be found in the [Examples folder of the Normalize NX-OS JSON Data Structures GitHub repository](https://github.com/ChristopherJHart/normalize-nxos-json-data-structures/blob/main/examples/netmiko_eigrp_neighbors.py). The corresponding unit tests for this script can be found in the [Tests folder of the same repository](https://github.com/ChristopherJHart/normalize-nxos-json-data-structures/blob/main/tests/test_examples/test_netmiko_eigrp_neighbors.py).

### Example Two - JSON Output From Scrapli

The business requirements provided to you for this automation restricts you from gathering this information via NX-API or SNMP, but allows access to the switch via SSH. Your boss also dislikes open-source software [created by people named Kirk, for some odd reason](https://pynet.twb-tech.com/).

An example of the Python script you might build for this task can be found in the [Examples folder of the Normalize NX-OS JSON Data Structures GitHub repository](https://github.com/ChristopherJHart/normalize-nxos-json-data-structures/blob/main/examples/scrapli_eigrp_neighbors.py). The corresponding unit tests for this script can be found in the [Tests folder of the same repository](https://github.com/ChristopherJHart/normalize-nxos-json-data-structures/blob/main/tests/test_examples/test_scrapli_eigrp_neighbors.py).

### Example Three - On-The-Box Python

The business requirements provided to you for this automation require you to use an on-the-box Python script to gather this information, as you are not allowed to use NX-API, SNMP, or SSH to access the switch.

An example of the Python script you might build for this task can be found in the [Examples folder of the Normalize NX-OS JSON Data Structures GitHub repository](https://github.com/ChristopherJHart/normalize-nxos-json-data-structures/blob/main/examples/on_box_eigrp_neighbors.py). The corresponding unit tests for this script can be found in the [Tests folder of the same repository](https://github.com/ChristopherJHart/normalize-nxos-json-data-structures/blob/main/tests/test_examples/test_on_box_eigrp_neighbors.py).
