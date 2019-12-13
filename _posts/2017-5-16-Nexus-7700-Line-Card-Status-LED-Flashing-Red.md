---
layout: post
title: Cisco Nexus 7700 – Line card’s Status LED flashing red instantly on insertion
---

A few weeks ago, a coworker of mine asked for my assistance with an odd Nexus 7700 issue. He brought me over to an N77-C7702, where I found a SUP2E humming along while an N77-F348XP-23 line card sat above it. The line card’s Status LED was blinking red, indicating its unhappiness with the current situation.

![]({{ site.baseurl }}/images/2017-5-Nexus-7702.jpg)

I consoled into the device, where a `show module` command revealed that the supervisor did not detect anything inserted into the chassis’s only slot. A quick re-seat of the line card resulted in the line card’s Status LED instantly switching back to a flashing red, with no change in the output of the `show module` command and not even so much as a message in the log. I checked the usual suspects, confirming that the software version and supervisor are compatible with the line card, verifying that the chassis had enough available power to power on the module, and inspecting the card for any obvious physical damage. I also modified the boot statement to reload into a newer version of software, just in case we were running into an odd bug on that particular software version. Everything checked out, so we were beginning to suspect the line card.

We removed the line card and found an F2 card to try instead. Much to our dismay, it gave us the exact same issue: a flashing red Status LED instantly on insertion. We also inserted the original F3 line card into another chassis, and it powered on without an issue. Thanks to the process of elimination, this instantly pointed the finger at one of two things: either there was an issue with the chassis, or there was an issue with the supervisor. Unfortunately, we didn’t have another chassis to test the issue with, but we did have another SUP2E. We swapped out the supervisors, and shortly after the supervisor finished booting, the module came online!

At that point, my coworker was convinced we had a faulty supervisor, but I remained skeptical. Something just didn’t seem right; the supervisor was able to boot up without reporting any critical errors, had no issues accepting any configuration, but wasn’t able to power on any module?

While absentmindedly staring at the console while the supervisor was booting, I noticed the following error message scroll by:

```
<<%PLATFORM-1-PFM_ALERT>> Incompatible Sup FPGA(13), upgrade FPGA >= 0x14
```

“Now *that* is unusual”, I thought to myself. A Google search led me to the [Cisco Nexus 7000 NX-OS 7.2 release notes](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/sw/7_x/nx-os/release/notes/72_nx-os_release_note.html), which contained the following statement:

* All supervisor 2E modules shipped with the Nexus 7702 switch will be shipped with FPGA version 1.4.
  * If you install a spare Supervisor 2E module on the Nexus 7702 switch you must upgrade the FPGA version to 1.4.
  * In such a situation you will be notified with alert: “<<%PLATFORM-1-PFM_ALERT>> Incompatible Sup FPGA(12), upgrade FPGA >= 0x14 “.

After doing a bit more research as to what exactly an FPGA is (as well as what an EPLD is, as the two are closely related) I determined that outdated FPGA firmware was likely causing the issues we had been experiencing. I followed the procedure highlighted in the [FPGA/EPLD upgrade release notes](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/sw/7_x/epld/epld_rn_7x.html), and about fifteen minutes later, the original F3 line card had powered on without an issue!

A mentor of mine once said that one of the beautiful aspects of Cisco’s Nexus devices is that they will always tell you exactly why something is not working *somewhere*; the trick is finding out where!