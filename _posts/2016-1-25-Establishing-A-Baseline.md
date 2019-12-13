---
layout: post
title: Establishing A Baseline
---

Lao Tzu, a Chinese philosopher, once said that “the journey of a thousand miles starts with a single step.” Although the journey of my homelab started over a year ago, it has remained largely undocumented. Blog posts like these (in combination with an internal wiki that I have set up) strive towards resolving that issue, as well as allowing others a glimpse into the infrastructure I have created and learn something from it (or, as is more likely, tell me that what I’m trying to do is very, *very* wrong.)

To start off, I’d like to preface this post by saying that I have absolutely no idea what I am doing, and that I am fully aware of it. Part of the reason that I want to document the baby steps that I am taking is so that in the next few years, I can look back at where I started and realize exactly how far I have come. I *will*  make incredibly stupid mistakes both large and small, and chances are very high that I will not realize that these mistakes exist until somebody else points them out.

With that being said, let’s take a look at my lab’s current topology.

![]({{ site.baseurl }}/images/2016-1-Lab-Topology-Censored.jpg)

Not pictured in this diagram is a Cisco 3825 router and a Cisco WS-3750-24T-S (24-port gigabit L3 managed switch), both of which I have recently purchased and will be configuring next week. I’m very excited about acquiring these guys off of eBay, as they’ll let me clean up my network quite a bit by eliminating both TP-Link routers, the TP-Link switch, and by allowing me to easily virtualize the PLEX server and repurpose the media server for something else (something that I have technically been able to do ever since I acquired the Amazon Fire TV around Christmas.) It will also be my first opportunity to extensively play around with VLANs, and it will give me experience with Cisco equipment that I can apply towards a certification sometime in the near future.

The pfSense router virtual machine acts as a virtual router, separating the rest of the virtual machines from the TEST01 virtual machine. I am currently reading [Learn Windows PowerShell in a Month of Lunches by Jeffery Hicks and Don Jones](https://www.amazon.com/Learn-Windows-PowerShell-Month-Lunches/dp/1617291080), which recommended following along with the book and completing lab exercises in a Server 2008 R2 environment configured as a primary domain controller. At the time, I was unsure what kind of configuration changes the book would have me practice, so I figured it would be best to work in an environment separate from the rest of the homelab. Placing the virtual machine behind a virtual router was a cheap and effective way to create this segregation.

The Dell PowerEdge 2950 is not currently in use at the moment primarily because of the sheer amount of noise it produces within my apartment; I also don’t need it at the moment, as I’m not anywhere close to overloading the much-quieter PowerEdge 1950 just yet. I will most likely be using the 2950 to play around with Hyper-V in the future, which means it will be powered on and screaming only when I am actively using it. I originally planned to use it as the second half of an ESXi cluster in conjunction with the existing 1950 , but I think I will choose to purchase one or two R610 or R710s to fulfill that role.

Speaking of, I should probably list the specific components of the 1950 (I won’t list the specs of the 2950, since I’m not entirely sure what’s in it and I don’t plan on using it anytime soon). It is a Dell PowerEdge 1950 Gen. II that was manufactured in late 2007, according to the service tag. It is equipped with the following:

* **CPU:** 2x Intel Xeon E5345 @ 2.33GHz
* **Memory:** 32GB DDR2 ECC
* **Storage:** 4x 500GB 7200RPM SATA hard drives configured in RAID10 via a PERC5/i RAID controller
* **Network:** 2x GbE network adapters
* **Power:** 2x 670W redundant power supplies
* **Remote Access:** 1x iDRAC5 card

I purchased the 1950 and the 2950, along with their respective rails, for $100 each on Craigslist, and the seller was kind enough to throw in the 500GB hard drives with the 1950 for an additional $20. All in all, I don’t think it was a bad deal, considering it was my first server.

Both servers (as well as the new Cisco equipment) are currently racked in an HP S10614, which is a 14U rack that I purchased very recently on Craigslist. It’s the perfect height for my equipment so far, giving me a decent amount of room to expand in the future and fits snugly in my office. Conveniently, it is almost exactly the same height as my desk, which means it can (and currently is) serving as a side table in addition to holding the majority of my equipment. I plan on taking pictures of my equipment before and after I configure the Cisco router and switch, just to show how much cleaner a rack and some proper gear can make a homelab look and function.

That’s all for now! Let me know if you have any comments, questions, or (most importantly) suggestions!