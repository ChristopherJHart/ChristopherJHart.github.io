---
layout: post
title: CCNP ROUTE Lab - EIGRP, DMVPN, PBR, and PAT
---

During my CCNP Route studies, I created a lab involving the configuration and manipulation of EIGRP, DMVPN, PBR, and PAT. The topology and instructions are as follows:

![]({{ site.baseurl }}/images/2018-4-CCNP-Route-Topology.png)

Both VIRL and GNS3 project files are available for download below. The pre-existing configurations on devices include loopback interfaces addressed with /24s (the /16s in the topology are actually multiple /24 loopbacks, allowing for the user to practice summarization.)

The VIRL project file can be [downloaded and imported here]({{ site.baseurl }}/files/2018-4-CCNP-Route-Lab-VIRL-Project.zip). You can import the file in VM Maestro by going to File, Import, then selecting Archive File under the General folder.

The GNS3 project file can be [downloaded and imported here]({{ site.baseurl }}/files/2018-4-CCNP-Route-Lab-GNS3-Project.tar). The topology utilizes the c3725-adventerprisek9-mz.124-15.T14.bin image file, but the project file can easily be edited in a text editor (such as Notepad++) to use your preferred image file.

I hope this helps people with their studies!