---
layout: post
title: Block/Column Select for Network Operators, NetDevOps, and Network Automation Developers
---

One of the most powerful, but underrated tools available to network operators is a feature called "block select" (sometimes also called "column select") supported by most text editors, such as Notepad++, Visual Studio Code, and Sublime Text. This feature allows you to select a block of text, rather than a continuous line of text. It also enables you to edit the block of text as a single unit. This is extremely useful when working with network device configurations, `show` command output, and other network data.

The keyboard shortcut for block select may vary from one text editor to another, but generally for Windows systems, it is activated with `Shift` + `Alt`. On Mac systems, it is `Shift` + `Option`. With these keys pressed, you can use the mouse to select a block of text. You can also use the arrow keys to select a block of text.

The process and utility of the block select is best demonstrated through practical examples, so the remainder of this post will walk through some realistic use cases in Visual Studio Code.

## Selecting a List of Interfaces

Let's say you need a list of all interfaces that are in an up/up state on a switch or router. Each interface must be on its own line so that you can easily paste it into another application, like a spreadsheet in Excel. Some example output is below.

```
C3560-CX#show ip interface brief | include up
Vlan1                  unassigned      YES NVRAM  up                    up
Vlan10                 192.168.10.3    YES NVRAM  up                    up
GigabitEthernet0/1     unassigned      YES unset  up                    up
GigabitEthernet0/2     unassigned      YES unset  up                    up
GigabitEthernet0/3     unassigned      YES unset  up                    up
GigabitEthernet0/4     unassigned      YES unset  up                    up
GigabitEthernet0/5     unassigned      YES unset  up                    up
GigabitEthernet0/6     unassigned      YES unset  up                    up
GigabitEthernet0/12    unassigned      YES unset  up                    up
GigabitEthernet0/13    unassigned      YES unset  up                    up
GigabitEthernet0/14    unassigned      YES unset  up                    up
```

You *could* manually copy and paste the name of each interface into the application, but that would be tedious and error-prone, especially if the network device has several dozen or hundreds of interfaces (such as a core switch with many SVIs [Switched Virtual Interfaces] or a router with many subinterfaces).

A NetDevOps or software developer might feel the itch to create a Python script to parse this information. Existing tools like Cisco's [pyATS/Genie parsers](https://pubhub.devnetcloud.com/media/genie-feature-browser/docs/#/parsers), Google's [TextFSM](https://github.com/google/textfsm) combined with Network to Code's [existing library of TextFSM templates](https://github.com/networktocode/ntc-templates), or even just plain old regular expression (regex) can be used to parse this information. If you needed to perform this task at scale across several dozen or hundreds of devices, that would indeed be the correct route to take! But if you just need to do this once or twice, developing a one-off Python script may feel excessive, even if it only takes 5 or 10 minutes.

This is where the block select feature shines! We can parse out the list of interfaces in a matter of seconds, without writing a single line of code.

After copying the output to a text editor of our choice, we left click at the beginning of the first line of the output, then hold down `Shift` and `Alt` and left click at the beginning of the last line of the output. This expands our cursor to encompass multiple lines of text.

![]({{ site.baseurl }}/images/2023/block-select/up-up-interface-1.apng)

From here, if we use our left and right arrow keys, we can move the entire expanded cursor through the output from the network device.

![]({{ site.baseurl }}/images/2023/block-select/up-up-interface-2.apng)

We can also use the `Home` and `End` keys to move the expanded cursor to the beginning and end of the line.

![]({{ site.baseurl }}/images/2023/block-select/up-up-interface-3.apng)

Let's cancel our block select with the `Escape` key, then move our cursor to the end of the first line. From here, we will hold down `Shift` and `Alt` and left click at the beginning of the IPv4 address column on the last line of the output. This will expand our cursor to encompass all of the columns of text that we're *not* interested in. Then, we can remove all of this text by pressing the `Delete` key.

![]({{ site.baseurl }}/images/2023/block-select/up-up-interface-4.apng)

We're not quite done yet - there's multiple blank spaces at the end of each line, which Visual Studio Code conveniently highlights for us. With our expanded cursor, we can select these blank spaces with the left and right arrow keys while the `Control` and `Shift` keys are held down. Then, we can remove these blank spaces by pressing the `Delete` key.

![]({{ site.baseurl }}/images/2023/block-select/up-up-interface-5.apng)

Finally, we have a newline-separated list of interfaces that are in an up/up state that can be easily copied and pasted into our application. Once you're familiar with the hotkeys, this whole process takes about 10-15 seconds to perform.

## Configuring a List of Interfaces

For our next use case, let's say we need to add one or more lines of configuration to multiple interfaces on a network device. For example, let's say you need to add the `no ip redirects` command to several dozen or hundred SVIs on a core switch to disable ICMP redirects. Configuring this manually would be painful, and even if you wrote a Python script to do it, it would take longer than the block select method.

```
C3560-CX#show running-config | include ^interface Vlan
interface Vlan1
interface Vlan10
interface Vlan11
interface Vlan12
interface Vlan13
interface Vlan14
interface Vlan15
interface Vlan16
interface Vlan17
interface Vlan18
interface Vlan19
interface Vlan20
interface Vlan21
interface Vlan22
interface Vlan23
interface Vlan24
interface Vlan25
interface Vlan26
interface Vlan27
interface Vlan28
interface Vlan29
interface Vlan30
```

After copying the output to a text editor of our choice, we left click at the beginning of the first line of the output, then hold down `Shift` and `Alt` and left click at the beginning of the last line of the output. This expands our cursor to encompass multiple lines of text. Then, we hit the `End` key to move the expanded cursor to the end of each line of interface configuration.

![]({{ site.baseurl }}/images/2023/block-select/interface-configuration-1.apng)

Next, if we hit the `Enter` key, we can see that a new line is inserted in between each line of interface configuration. Furthermore, our expanded cursor now starts on each new line of the configuration! This is exactly what we want, because we can now add the `no ip redirects` command to each interface configuration by simply typing it in.

![]({{ site.baseurl }}/images/2023/block-select/interface-configuration-2.apng)

From here, we can copy and paste the resulting commands into our network device. Again, once you're familiar with the hotkeys, this process takes seconds, which is significantly better than the alternative of manually typing in the command on each interface or developing a Python script.

## Conclusion

Needless to say, the block select feature is a little-known, but exceptionally useful tool for network operators, NetDevOps, and network automation developers. It can be used to quickly parse and manipulate network device configurations, `show` command output, and other network data in ad-hoc scenarios where developing a Python script may not be appropriate. If you're not already using the block select feature in your daily work, I highly recommend that you start!

## References

* [Column (box) selection in Visual Studio Code](https://code.visualstudio.com/docs/getstarted/tips-and-tricks#_column-box-selection)
* [Column Selection in Sublime Text](https://www.sublimetext.com/docs/column_selection.html)
