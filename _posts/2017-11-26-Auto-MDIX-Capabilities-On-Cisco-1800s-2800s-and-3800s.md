---
layout: post
title: Auto-MDIX Capabilities on Cisco 1800s, 2800s, and 3800s
---

A common question I have seen asked is what models of Cisco 1800s, 2800s, and 3800s support auto-MDIX, a feature that allows for Ethernet cables with almost any pinout to connect devices together. This question has become prevalent now that these devices are becoming more affordable on the used Cisco market, making them especially prominent in home or business lab environments. I have personally tested all of the below devices to verify the presence or absence of auto-MDIX.

| Model     | Auto-MDIX |
|-----------|-----------|
| CISCO1801 | Yes       |
| CISCO1802 | Yes       |
| CISCO1803 | Yes       |
| CISCO1811 | Yes       |
| CISCO1812 | Yes       |
| CISCO1841 | No        |
| C1861     | Yes       |
| CISCO2801 | No        |
| CISCO2811 | Yes       |
| CISCO2821 | Yes*      |
| CISCO2851 | Yes*      |
| CISCO3825 | Yes*      |
| CISCO3845 | Yes*      |

*These devices have Gigabit Ethernet interfaces, which support auto-MDIX by default according to IEEE 802.3-2012, section 40.8.2

## Testing Procedure

The testing procedure for each of these devices was fairly straightforward. I created a loopback by inserting a crossover Ethernet cable into both onboard interfaces of each device (for example, Fa0/0 and Fa0/1). I would verify that both interfaces are functional and come up/up by issuing a `no shutdown` command. Then, I would replace the crossover Ethernet cable with a straight-through Ethernet cable. If the interfaces were functional and came up/up, then the device supported auto-MDIX. If the interfaces remained offline in a down/down state, then the device did not support auto-MDIX.

In some instances, such as the CISCO1801 and the C1861, the device only had a single onboard Ethernet interface. In these cases, I repeated the test using another device that does not support auto-MDIX, such as a CISCO1841. In order for two devices of similar type (for example, two routers) to connect interfaces via a straight-through cable, only one device in the pair needs to support auto-MDIX. Therefore, I would verify the interfaces on both devices were working using a crossover Ethernet cable, then switch to a straight-through Ethernet cable. If the link came online, then through the power of elimination, the device under test is confirmed to support auto-MDIX. If the link did not come online with a straight-through Ethernet cable, then neither device supported auto-MDIX.

Hopefully this helps quell some confusion regarding auto-MDIX support on these models!