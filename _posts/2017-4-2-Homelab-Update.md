---
layout: post
title: A Homelab Update!
---

A lot has changed with my homelab since my first post, and unfortunately, my website has remained devoid of any updates. That is definitely going to change during 2017! To start off, let’s take a look at the latest lab topology, then I’ll explain what has changed since the previous update and my rationale behind certain decisions.

![]({{ site.baseurl }}/images/2017-4-Homelab-Update.jpg)

First, I’ll talk about things from a networking point of view. My TP-Link routers and switches have been replaced with a Cisco 3825 router and a Cisco 3750G (specifically, a WS-C3750G-24T-S, or for those who don’t speak Cisco model numbers, a 24-port gigabit L3 switch). I have also purchased an Ubiquiti UniFi AP AC Lite to serve as my access point. My “production” network and my lab network are no longer segregated the way they used to be, as I felt comfortable enough with Cisco devices and my own technical abilities to merge the two together. The introduction of the Cisco switch and router allowed me to introduce VLANs to the network.

Initially, I went a little bit overboard with VLANs. I introduced a “primary clients” VLAN, a wireless VLAN, a guest wireless VLAN, a “devices” VLAN (which I primarily put my printer on), a services VLAN, a management VLAN, a monitoring VLAN, a DMZ VLAN, and a VLAN for a test PowerShell environment. Each VLAN received its own /24 network, which was subnetted from a 10.0.0.0/8. After realizing that some VLANs were not being used, were unnecessary, and were complicating my IP addressing scheme, I recently consolidated down to 5 VLANs, 4 of which are displayed in the topology.

* **VLAN 10**  – designated for “clients”, which include all wired and wireless devices (printer included).
* **VLAN 15** – designated for guest wireless traffic, tagged at the Ubiquiti AP.
* **VLAN 20** – designated for management (primarily of the router/switch, VCSA, my ESXi host, and the iDRAC of my server).
* **VLAN 30** – designated for services (effectively merging the services and monitoring VLANs).
* **VLAN 40** – designated as the DMZ, which is segregated from the rest of the network via an ACL on the router.

Each of these VLANs is still given its own /24, but is being subnet from 192.168.0.0/16, with the third octet of the IP address designating the VLAN it belongs to. For example, devices on VLAN 10 live on the 192.168.10.0/24 network.

DHCP and DNS are being served by the domain controllers on my network. DC01 and DC03 are configured for failover DHCP (which lets me reboot one of them to update without disturbing network services.) DC02 is not configured for DHCP; this is because DC01 and DC02 are virtualized, but DC03 is a physical machine (an old Toshiba Satellite humming along on my desk). My rationale for this decision was to ensure availability of network services should the ESXi host go down, whether it be due to a prolonged power outage or simply for maintenance.

Moving onto the server hardware changes, you will notice that both the old PowerEdge 1950 and 2950 have been decommissioned. I finally grew weary of the noise and power consumption of the 1950 and purchased a shiny used PowerEdge R710. This R710 is equipped with two Intel Xeon L5640s (picked out with power consumption in mind), as well as 64GB of memory (a decent upgrade from my previous 32GB, which was beginning to fill up.) During the migration between the two servers, I decided to update my version of ESXi from 6.0 to 6.5.0, giving me access to the vSphere HTML5 client (which is very nice!)

So far, I am enjoying the R710. I would not say that the server is necessarily quieter than the 1950, but the R710’s fans run at a lower pitch than the 1950’s fans, which effectively makes it sound a bit quieter. The power consumption is where the differences become massive; my entire homelab (router and switch included) runs at about 120W, whereas with the 1950 it would run anywhere from 300-350. This has put my noticeable dent in my monthly power bill, so I think the upgrade was well worth the money!

Next, let’s take a look at the virtual machines. Compared to my last topology’s count of 6, it is clear that I have drastically increased the number of VMs I host (currently at 14 worth mentioning on the topology – I have a few others that are temporary or not noteworthy.)

* **PLEX01** – I have virtualized my PLEX server and have it running under CentOS 7 – previously, APP01 was serving as my PLEX server in addition to a number of other functions, but I decided to move PLEX off of a Windows platform to alleviate some performance issues that I was encountering. So far, the only downside to hosting PLEX on Linux is that updating PLEX is not as easy as clicking a button; however, I believe that it would be fairly easy to script, so I plan on implementing that sometime Soon(TM).

* **GNS01** – This is a VM I use to host all of my running router images on GNS3. This will come in especially handy as I approach my CCNP studies during this year.

* **APP01** – This is a Server 2012 VM that runs a few little pieces of Windows-based software that are not resource-intensive, such as Progress Quest, another (legal) bot for a video game I play that is similar to Progress Quest, and qBitTorrent (which I plan on replacing with rTorrent on TRRNT01 Soon(TM)).

* **PROXY01** – This is an HAProxy instance that functions as a reverse proxy, dividing incoming web traffic to different web servers depending on what website it’s headed to. HAProxy was recommended by a friend of mine for its ease of use, and it is incredibly powerful in addition to being easy-to-use.

* **LEMP01-03** – These LEMP stacks host various external and internal websites.

* **MON01** – This VM hosts Zabbix, which I use to monitor pretty much everything. For the most part, I use it to check up on how much bandwidth I’m using on my switch/router and help drill down what’s currently downloading something, but I intend on properly setting up triggers and alert notifications in the future. Zabbix is a bit of a beast to get configured properly, but it has proven to be extremely capable and flexible in its abilities.

* **OWNCLOUD01** – This VM hosts OwnCloud, which I haven’t quite gotten working yet. I plan on using it to share files and such with third-parties, as well as serve as a central repository for software downloads for myself.

* **MINE01** – This hosts a Minecraft server for myself, friends, and family to use every once in a while. I originally dabbled with using MineOS to accomplish this task, but it didn’t play very nicely with third-party mods, so I dumped it for CentOS in favor of more flexibility.

* **TKT01** – I plan on setting this VM up with Request Tracker soon, which will be used as my to-do list for homelab and home automation tasks.

Moving on to non-homelab topics, I recently changed jobs and now work at Cisco’s RTP campus as a co-op in the CALO labs! This has been an extraordinary opportunity, and I have learned an incredible amount. My goal when working in IT has always been to make sure that the present-time Chris believes that the Chris who was working a month ago is an idiot, and Cisco has absolutely provided that for me. I not only get the opportunity to work with extremely expensive equipment at a physical layer, but also have the chance to work with some of the best network engineers in the business!

As you can imagine, Cisco pushes all of its technical employees to obtain network certifications, and co-ops are no different. We were encouraged to obtain our CCNA R&S by the end of June, and I am pleased to say that I obtained mine on March 20th! Next, I plan on working towards my CCNA Security before strengthening my routing/switching skills with a CCNP R&S.

That’s all for now! With any luck, I’ll make enough progress and have enough time to post more updates within the next few days! As always, if you have any questions, comments, or suggestions, feel free to let me know!