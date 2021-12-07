---
layout: post
title: Quick Mystery - Spanning Tree Priority
---

One aspect of Spanning Tree Protocol I became curious about during my CCIE studies was the exact behavior behind how Spanning Tree bridge priorities are modified using the `spanning-tree vlan {vlan-id} root {primary | secondary}` configuration command. This post explores this behavior in the lab!

## Mystery

In this post, we seek to answer two questions:

1. If a bridge is configured to automatically become the root of the Spanning Tree with the `spanning-tree vlan {vlan-id} root primary` configuration command, will it preempt itself as the root if another bridge’s priority suddenly becomes lower and challenges the root bridge?

2. What is the exact behavior behind the `spanning-tree vlan {vlan-id} root {primary | secondary}` configuration command?

## Topology

![]({{ site.baseurl }}/images/2019-11-QM-STP-Topology.png)

Testing was performed through VIRL 1.6.0 using IOSvL2 nodes running IOS 15.2.

## Testing

First, let’s check the current Spanning Tree state in VLAN 1 on S1. By default, S1 is the root bridge because it holds the lowest bridge address in the L2 domain.

```
S1#show spanning-tree vlan 1
<snip>
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    32769
             Address     5e00.0000.0000
             This bridge is the root    <<<
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec


  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     5e00.0000.0000
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec
```

S1 is configured to automatically become the root of the L2 domain in VLAN 1 through the `spanning-tree vlan 1 root primary` configuration command.

```
S1#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
S1(config)#spanning-tree vlan 1 root primary
S1(config)#end
```

Let’s review the configuration that is *actually* applied.

```
S1#show run | i spa
spanning-tree mode pvst
spanning-tree extend system-id
spanning-tree vlan 1 priority 24576    <<<
```

Let’s verify that the bridge priority of S1 has changed accordingly.

```
S1#show spanning-tree vlan 1
<snip>
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    24577    <<<
             Address     5e00.0000.0000
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec


  Bridge ID  Priority    24577  (priority 24576 sys-id-ext 1)
             Address     5e00.0000.0000
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec
```

For future reference, note that the priority of 24,576 that was applied is 8192 less than the default bridge priority value of 32,768. Next, let’s configure S2 with a bridge priority of 16,384.

```
S2#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
S2(config)#spanning-tree vlan 1 priority 16384
S2(config)#end
```

S2 confirms that it is now the root bridge.

```
S2#show spanning-tree vlan 1
<snip>
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    16385    <<<
             Address     5e00.0001.0000
             This bridge is the root    <<<
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec


  Bridge ID  Priority    16385  (priority 16384 sys-id-ext 1)
             Address     5e00.0001.0000
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  15  sec
```

This suggests that S1 does not preempt itself as the root bridge when configured with `spanning-tree vlan {vlan-id} root primary`. We can confirm this by checking the Spanning Tree state and configuration of S1.

```
S1#show spanning-tree vlan 1
<snip>
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    16385
             Address     5e00.0001.0000    <<<
             Cost        4
             Port        2 (GigabitEthernet0/1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec


  Bridge ID  Priority    24577  (priority 24576 sys-id-ext 1)    <<<
             Address     5e00.0000.0000
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  15  sec

S1#sh run | i spa
spanning-tree mode pvst
spanning-tree extend system-id
spanning-tree vlan 1 priority 24576    <<<
```

The results of this test answer our first question. **A switch configured with `spanning-tree vlan {vlan-id} root primary` will not preempt the root bridge role if another bridge with a superior BPDU usurps the role**. Instead, this configuration command is essentially a macro that causes the local switch to analyze the current root bridge’s ID, determine what bridge priority configuration needs to be applied to become the root bridge, and apply the resulting configuration. This macro only runs when the `spanning-tree vlan {vlan-id} root primary` command is executed, so it is not able to preempt the root bridge role.

Now, let’s try a second scenario. What happens if we configure `spanning-tree vlan 1 root primary` once more on S1?

```
S1#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
S1(config)#spanning-tree vlan 1 root primary 
S1(config)#end
S1#show run | i spa
spanning-tree mode pvst
spanning-tree extend system-id
spanning-tree vlan 1 priority 16384
S1#show spanning-tree vlan 1
<snip>
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    16385
             Address     5e00.0000.0000
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec


  Bridge ID  Priority    16385  (priority 16384 sys-id-ext 1)
             Address     5e00.0000.0000
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  15  sec
```

S1 analyzed the current root bridge’s ID and found that it could become the root bridge if it lowered its priority to match that of the existing root bridge (S2) because its system MAC address (5e00.0000.0000) is lower than S2’s (5e00.0001.0000).

At first glance, this behavior seems contrary to what is stated in Cisco’s documentation. The [“Configuring STP” chapter of the Catalyst 3750 Switch Software Configuration Guide ](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3750/software/release/15-0_2_se/configuration/guide/scg3750/swstp.html#23902) for IOS 15.0(2)SE and Later states the following:

> "To configure a switch to become the root for the specified VLAN, use the `spanning-tree vlan vlan-id root` global configuration command to modify the switch priority from the default value (32768) to a significantly lower value. **When you enter this command, the software checks the switch priority of the root switches for each VLAN.** Because of the extended system ID support, **the switch sets its own priority for the specified VLAN to 24576 if this value will cause this switch to become the root for the specified VLAN.**

> If any root switch for the specified VLAN has a switch priority lower than 24576, the switch sets its own priority for the specified VLAN to 4096 less than the lowest switch priority."

The first paragraph describes the behavior we initially observed when we configured `spanning-tree vlan 1 root primary` on S1 for the first time. In that scenario, the bridge priority of all bridges was set to 32,768, so S1 set its bridge priority to 24,576 as described by the documentation.

However, the second paragraph describes behavior that S1 did not abide by in the second scenario. In the second scenario, the existing root bridge’s priority was 16,384. According to the documentation, S1 *should* have set its bridge priority to 12,288. Instead, **S1 knew that its system MAC address was low enough to become the root bridge if it matched S2’s bridge priority, so it modified its configuration accordingly.**

Let’s try a third scenario involving S4, which has a system MAC address of 5e00.0003.0000. In this scenario, all bridges started with a bridge priority of 32,768. S1 was configured with `spanning-tree vlan 1 root primary`, applying a configuration of `spanning-tree vlan 1 priority 24576` to the device.

```
S1#show run | i spa
spanning-tree mode pvst
spanning-tree extend system-id
spanning-tree vlan 1 priority 24576    <<<
S1#show spanning-tree vlan 1
<snip>
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    24577    <<<
             Address     5e00.0000.0000
             This bridge is the root    <<<
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec


  Bridge ID  Priority    24577  (priority 24576 sys-id-ext 1)
             Address     5e00.0000.0000
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  15  sec
```

S4 agrees that S1 is currently the root bridge.

```
S4#sh spanning-tree vlan 1
<snip>
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    24577    <<<
             Address     5e00.0000.0000    <<<
             Cost        4
             Port        6 (GigabitEthernet1/1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec


  Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
             Address     5e00.0003.0000
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  15  sec
```

Let’s configure S4 with the `spanning-tree vlan 1 root primary` configuration command and see what configuration is applied.

```
S4#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
S4(config)#spanning-tree vlan 1 root primary 
S4(config)#end
S4#sh run | i spa
spanning-tree mode pvst
spanning-tree extend system-id
spanning-tree vlan 1 priority 20480
```

Finally, let’s confirm that S4 now views itself as the root bridge.

```
S4#show spanning-tree vlan 1
<snip>
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    20481
             Address     5e00.0003.0000
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec


  Bridge ID  Priority    20481  (priority 20480 sys-id-ext 1)
             Address     5e00.0003.0000
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec
```

Here, S4 recognized that if it matched S1’s bridge priority of 24,576, it would still lose the root bridge election because S1’s system MAC address is lower (and, therefore, superior) to S4’s own system MAC address. In order to honor the administrator’s intent of making S4 the root bridge, **S4 lowered its own bridge priority by 4096 to 20,480 as described by the documentation mentioned above.**

Finally, let’s give S3 some attention by configuring it as a secondary root bridge with the `spanning-tree vlan 1 root secondary` configuration command. First, let’s verify S3’s current perspective on the VLAN 1 spanning tree.

```
S3#show spanning-tree vlan 1
<snip>
VLAN0001
   Spanning tree enabled protocol ieee
   Root ID    Priority    20481
              Address     5e00.0003.0000
              Cost        4              
              Port        2 (GigabitEthernet0/1)              
              Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec   


   Bridge ID  Priority    32769  (priority 32768 sys-id-ext 1)
              Address     5e00.0002.0000
              Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
              Aging Time  300 sec
```

Recall that while there can only be a single root bridge in an L2 domain, the administrator can configure multiple bridges as “secondaries” to act as backup root bridges. Bridges configured with the `spanning-tree vlan {vlan-id} root secondary` configuration command will be *more likely* (but **not** guaranteed, as our scenario will show) to become the root bridge in the event that the primary root bridge fails.

Let’s configure S3 with the `spanning-tree vlan {vlan-id} root secondary` command and see what configuration is applied.

```
S3#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
S3(config)#spanning-tree vlan 1 root secondary
S3(config)#end
S3#show run | i spa
spanning-tree mode pvst
spanning-tree extend system-id
spanning-tree vlan 1 priority 28672    <<<
```

As you can see, S3 modified its configuration such that its bridge priority is 28,672. This behavior is documented in the [“Configuring STP” chapter of the Catalyst 3750 Switch Software Configuration Guide for IOS 15.0(2)SE and Later](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3750/software/release/15-0_2_se/configuration/guide/scg3750/swstp.html#21421):

> "**When you configure a switch as the secondary root, the switch priority is modified from the default value (32768) to 28672**. The switch is then likely to become the root switch for the specified VLAN if the primary root switch fails. This is assuming that the other network switches use the default switch priority of 32768 and therefore are unlikely to become the root switch."

However, in the event that the root bridge (S4) fails, S3 will not become the root bridge, because S1 still has a lower priority.

```
S1#show spanning-tree vlan 1
<snip>
VLAN0001
  Spanning tree enabled protocol ieee
  Root ID    Priority    20481
             Address     5e00.0003.0000
             Cost        4
             Port        6 (GigabitEthernet1/1)
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec


  Bridge ID  Priority    24577  (priority 24576 sys-id-ext 1)    <<<
             Address     5e00.0000.0000
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  300 sec
```

As mentioned before, the `spanning-tree vlan {vlan-id} root secondary` command simply makes a bridge *more likely* to become the root bridge in the event that the primary root bridge fails. However, it does not guarantee what specific place in the hierarchy the bridge holds.

## Results

In conclusion, we’ve demonstrated three things:

1. A bridge configured as the primary root bridge through the `spanning-tree vlan {vlan-id} root primary` configuration command will not preempt the root bridge role if a bridge with a superior BPDU comes online and usurps the role.

2. The `spanning-tree vlan {vlan-id} root primary` configuration command can guarantee that the local bridge will become the root bridge *at the instantaneous moment that command is executed*. However, because it lacks any preempt behavior, there is no guarantee that the local bridge will *remain* the root bridge in the future.

3. The `spanning-tree vlan {vlan-id} root secondary` configuration command is only truly helpful if all other bridges (aside from the root bridge) are configured with a default bridge priority of 32,768. Otherwise, it may be best to manually configure a hierarchy of bridge priorities using the `spanning-tree vlan {vlan-id} priority {priority-value}` configuration command on your desired primary and secondary root bridges.

I hope that you find this helpful!