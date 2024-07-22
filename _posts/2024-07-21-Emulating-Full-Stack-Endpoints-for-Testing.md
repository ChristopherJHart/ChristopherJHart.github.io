---
layout: post
title: "Emulating Full Stack Endpoints for Testbeds with Virtualized Endpoints and PCIe Passthrough"
---

For the past few years, I have helped lead [Cisco's Solution Validation Services (SVS) team](https://www.cisco.com/c/en/us/solutions/collateral/executive-perspectives/technology-perspectives/network-transform-lab-valid-so.html) within our United States Public Sector department. Simply put, our daily work involves testing Cisco products and solutions in a lab environment by deploying them precisely how the customer would deploy them in their production environment. We then validate that the solution meets the customers requirements and works as expected.

A recent project tasked us with validating the functionality of both guest and sponsor portals in Cisco's Identity Services Engine (ISE) before and after an upgrade. In a typical production environment, these portals would be accessed by real wired or wireless endpoints; a user with a desktop or laptop would physically connect to the network, be granted access to the network through 802.1X or MAC Authentication Bypass (MAB), and interact with the guest and sponsor portals.

Although modern traffic generators (such as Ixia or Spirent) can emulate 802.1X supplicants and generate stateless traffic, the emulated endpoints are not "full stack" endpoints. They cannot interact with the guest and sponsor portals in the same way that a real-world endpoint can. This is because the ISE guest and sponsor portals are web-based interfaces that require user interaction (entering credentials, submitting form data by clicking buttons, etc.), and most traffic generators are not capable of interacting with web-based interfaces in this way.

## COTS Endpoint Challenges

This requirement necessitated the use of a real-world wired endpoint that could be granted access to the network through 802.1X or MAC Authentication Bypass (MAB) and interact with the guest and sponsor portals. Although using a commercial off-the-shelf endpoint (like a Dell or Lenovo laptop) was possible, doing so would have also introduced several challenges.

### On-Site Presence

In order for us to perform testing with endpoints or troubleshoot issues requiring manipulation of endpoints, at least one individual must be physically present at the endpoint’s location within our lab environment.

This has three problems:

* **Reduced Remote-Friendly Work** – Although *some* members of our team are relatively close to the office where the testbed is located, many are not. Furthermore, the team members assigned to this particular project are all remote. This means a team member close to the office (who is not assigned to the project and has their own, just-as-important projects to work on) must be dispatched to the testbed to assist with testing or troubleshooting.
* **Heightened Opportunity for Miscommunication** – When the on-site individual is instructed to perform an action on the endpoint as part of testing or troubleshooting, the instructions – however simple they may seem – may be misinterpreted. This can lead to confusion if the instructions are nuanced, require a specific order of operations, or if the results require detailed interpretation. This ultimately stems from a lack of visibility that the instructor has into how actions are performed on the endpoint by the on-site individual and what the results of those actions look like.
* **Hampered Collaboration** – The endpoint, along with the rest of the testbed, is in a noisy data center environment. When working with others to troubleshoot issues during live collaboration (e.g. a Cisco Webex or Microsoft Teams meeting), both the instructions given to the individual and the results of those actions reported by the on-site individual are delayed due to “travel time”. In other words, the individual must receive the instructions, walk into the data center, perform the actions and wait for the results, walk out of the data center, and report the results. This lengthy feedback cycle prolongs the time to troubleshoot issues and perform testing.

### Single-User Visibility

Commercial off-the-shelf endpoints are typically only remotely accessible through Remote Desktop Protocol (RDP) or Virtual Network Computing (VNC) protocols. The sessions created by these protocols offer single-user access, which means only a single user can view and control a session at a time. This leads to the same **Heightened Opportunity for Miscommunication** and **Hampered Collaboration** challenges described above.

### No Remote Out-of-Band/Console Access

Remote access to commercial-off-the-shelf endpoints via RDP or VNC protocols can negate [On-Site Presence](#on-site-presence) challenges. However, they also rely upon the endpoint staying powered on, functioning correctly, and remaining network accessible. Murphy’s law (“Anything that can go wrong will go wrong”) tends to attack these dependencies; laptop chargers disconnecting through accidental nudging, endpoints crashing, devices hibernating due to misconfigured power settings, and other such issues can disrupt testing with no remote troubleshooting options, requiring [On-Site Presence](#on-site-presence) to investigate and resolve.

Furthermore, the nature of a lab environment means that oftentimes, if something is not working, you are not sure if the root cause of the issue is with the endpoint, or with the network devices to which the endpoint is connected. It can take a lengthy amount of time for a remote worker to troubleshoot network devices, only to find out that the problem was with a simple issue on the endpoint that could have been resolved in seconds if they had remote access to the endpoint’s console.

### Inefficient Test Artifact Sharing

As part of the testing process, test artifacts (such as command line output, screenshots, etc.) must be gathered from the endpoint and placed into the customer-facing test report. This process is inefficient when using commercial off-the-shelf endpoints, as the test engineer must manually gather these artifacts from the endpoint, transfer them to some form of a removable storage device (like a USB drive), and then transfer them to their own computer for inclusion in the test report. This file transfer process is time-consuming and error-prone, as it requires the test engineer to manually gather and transfer the artifacts, which can lead to artifacts being missed or lost.

### Poor Test Engineer Experience Causes Low Morale

The above challenges with using commercial off-the-shelf endpoints, although minor, can add up to tangible delays when executing testing or troubleshooting issues. These challenges can make the test engineer's job more difficult, frustrating, and less rewarding. Simply put, they're not fun to work with. Maintaining a high quality-of-life for test engineers is important to ensure they remain engaged, motivated, and can do their best work.

## Solution: Virtualized Endpoints with PCIe Passthrough

We chose to balance the valid emulation of real-world wired endpoints while minimizing the negative impacts of commercial off-the-shelf endpoints by deploying a UCS server running VMware ESXi. This server has an Intel i350 Network Interface Card (NIC) physically inserted in the mLOM slot (Cisco product ID UCSC-MLOM-IRJ45). This NIC offers four 1000BASE-T ports that can be connected via Cat5e Unshielded Twisted Pair (UTP) cables.

This ESXi host has four Windows 10 virtual machines deployed on it:

* SITE-1-ENTERPRISE-ENDPOINT
* SITE-2-ENTERPRISE-ENDPOINT
* SITE-1-GUEST-ENDPOINT
* SITE-2-GUEST-ENDPOINT

ESXi is configured to expose a single unique port of the Intel NIC to each Windows 10 virtual machine through a feature called *PCIe Passthrough*.

![]({{ site.baseurl }}/images/2024/emulating-testbed-endpoints/vm_pcie_passthrough_nic.png)

This gives each virtual machine exclusive access to a true hardware NIC, as opposed to a virtual NIC that is subject to the rules of VMware’s virtual switching constructs (which *could* interfere with 802.1X authentication in ways that a commercial off-the-shelf endpoint would never experience).

Each of the four ports on the Intel i350 NIC inserted in the UCS server is then connected to a specific port of a specific device within the testbed:

* SITE-1-ENTERPRISE-ENDPOINT owns MLOM Port 1 with a MAC address of a03d.6f89.d349 and connects to SITE-1-SW1’s GigabitEthernet1/0/1
* SITE-2-ENTERPRISE-ENDPOINT owns MLOM Port 2 with a MAC address of a03d.6f89.d34a and connects to SITE-2-SW1’s GigabitEthernet1/0/1
* SITE-1-GUEST-ENDPOINT owns MLOM Port 3 with a MAC address of a03d.6f89.d34b and connects to SITE-1-SW1’s GigabitEthernet1/0/2
* SITE-2-GUEST-ENDPOINT owns MLOM Port 4 with a MAC address of a03d.6f89.d34c and connects to SITE-2-SW1’s GigabitEthernet1/0/2

During the 802.1X authentication process, each virtual machine receives IP addressing and default gateway information from the relevant DHCP server within the testbed. This ensures that most traffic is sent out of the Intel i350 NIC into the testbed.

```
PS C:\Users\Tester> hostname
SITE-1-GUEST-ENDPOINT
PS C:\Users\Tester> ipconfig
<snip>
Windows IP Configuration


Ethernet adapter pciPassthru0:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::702a:774b:410c:b2e0%15
   IPv4 Address. . . . . . . . . . . : 192.0.2.10
   Subnet Mask . . . . . . . . . . . : 255.255.252.0
   Default Gateway . . . . . . . . . : 192.0.2.1
```

Each virtual machine is also configured in ESXi with a second virtual NIC connected to the out-of-band management network facilitating remote access to all devices in the testbed.

![]({{ site.baseurl }}/images/2024/emulating-testbed-endpoints/vm_management_nic.png)

A static IPv4 address is configured on this secondary NIC, but **no default gateway information is configured for this NIC**.

![]({{ site.baseurl }}/images/2024/emulating-testbed-endpoints/vm_static_ip_settings.png)

Instead, a static route pointing to a remote subnet (10.255.0.0/24) via the default gateway for the out-of-band management network is configured.

```
PS C:\Users\Tester> hostname
SITE-1-GUEST-ENDPOINT
PS C:\Users\Tester> route print -4
===========================================================================
Interface List
 15...a0 3d 6f 89 d3 4b ......Intel(R) I350 Gigabit Network Connection
 10...00 50 56 bd 78 65 ......Intel(R) 82574L Gigabit Network Connection
  1...........................Software Loopback Interface 1
===========================================================================

IPv4 Route Table
===========================================================================
Active Routes:
Network Destination        Netmask          Gateway       Interface  Metric
          0.0.0.0          0.0.0.0        192.0.2.1       192.0.2.10     25
        192.0.2.0    255.255.252.0         On-link        192.0.2.10    281
       192.0.2.10  255.255.255.255         On-link        192.0.2.10    281
      192.0.2.255  255.255.255.255         On-link        192.0.2.10    281
        127.0.0.0        255.0.0.0         On-link         127.0.0.1    331
        127.0.0.1  255.255.255.255         On-link         127.0.0.1    331
  127.255.255.255  255.255.255.255         On-link         127.0.0.1    331
       10.255.0.0    255.255.255.0       10.254.0.1     10.254.0.223     35
       10.254.0.0    255.255.255.0         On-link      10.254.0.223    281
     10.254.0.223  255.255.255.255         On-link      10.254.0.223    281
     10.254.0.255  255.255.255.255         On-link      10.254.0.223    281
        224.0.0.0        240.0.0.0         On-link         127.0.0.1    331
        224.0.0.0        240.0.0.0         On-link        192.0.2.10    281
        224.0.0.0        240.0.0.0         On-link      10.254.0.223    281
  255.255.255.255  255.255.255.255         On-link         127.0.0.1    331
  255.255.255.255  255.255.255.255         On-link        192.0.2.10    281
  255.255.255.255  255.255.255.255         On-link      10.254.0.223    281
===========================================================================
Persistent Routes:
  Network Address          Netmask  Gateway Address  Metric
       10.255.0.0    255.255.255.0       10.254.0.1      10
===========================================================================
```

This 10.255.0.0/24 subnet contains jumpboxes used by our team for out-of-band connectivity to the testbed. Therefore, this static route configuration enables remote access via Remote Desktop Protocol (RDP) to each virtual machine (among other management uses, such as file sharing) without compromising traffic intended for the testbed’s in-band network. Put another way, this allows us to get remote access to the endpoint if we need it while still ensuring all other traffic is sent out of the Intel i350 NIC into the testbed, not our management network.

## Benefits of Virtualized Endpoints with PCIe Passthrough

This solution faithfully emulates the behavior of real-world wired endpoints while solving all the challenges with using commercial off-the-shelf endpoints in a test environment.

### Remote-Friendly Work

No individual is required to be on-site with the endpoints during testing or troubleshooting involving endpoints. On-site troubleshooting is limited to ensuring power and network cables are properly connected to testbed devices. These are simple tasks with no nuances or specific order of operations required to complete, and their results are easy to verify with minimal interpretation needed by either the instructor or on-site personnel.

Furthermore, remote personnel are empowered to autonomously perform desired actions on endpoints and interpret the results of those actions without the assistance of on-site personnel. No communication is required with on-site personnel, which means opportunities for miscommunication with on-site personnel are eliminated.

Finally, collaboration is unfettered by the travel time of on-site personnel, resulting in more efficient testing and improved time-to-resolution of issues involving endpoints.

### Improved Visibility

VMware’s ESXi offers a multiplexed console for virtual machines, which means multiple users can view the console of a virtual machine at the same time. This facilitates collaboration and ensures that testing or troubleshooting steps are easily visible to all participants, which subsequently improves the quality of testing.

### Remote Out-of-Band/Console Access

The virtual machine console offered by VMware’s ESXi allows for easy remote troubleshooting of endpoint virtual machines if they crash, become unreachable, or otherwise do not behave as expected. Furthermore, the Cisco Integrated Management Controller (CIMC) included in all Cisco UCS servers enables the remote troubleshooting of the ESXi host itself, further improving the level of remote troubleshooting offered by this endpoint solution. So long as the UCS server is connected to a power source and its CIMC port is connected to the network (which are both trivial problems for on-site personnel to solve), remote personnel can autonomously troubleshoot and resolve without needing an on-site presence.

### Enterprise-Class Hardware Redundancy

Emulating endpoints with enterprise-grade server hardware offers hardware redundancy advantages not found in commercial off-the-shelf endpoints. Redundant power supply units and hard disk drives backed by Redundant Array of Independent Disk (RAID) arrays ensure that simple hardware failures do not introduce immediate delays to testing.

## Conclusion

The use of virtualized endpoints with PCIe Passthrough in a test environment offers a solution that faithfully emjsonulates the behavior of real-world "full stack" wired endpoints while minimizing the negative impacts of using commercial off-the-shelf endpoints. This solution improves the quality of life for test engineers, reduces the time to troubleshoot issues, and improves the efficiency of testing. Although it does not scale as well as endpoints emulated by traditional traffic generators like Ixia or Spirent, it is a cost-effective and highly-convenient solution for test requirements that do not require high scalability to test a particular feature.
