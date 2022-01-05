---
layout: post
title: TAC Tales - Saving Configuration is for Quitters
---

In December of 2019, a customer opened a TAC case where the `copy running-config startup-config` command would hang and time out after several minutes on a few brand-new Nexus switches. The command returned a "Configuration update aborted: timed out" error message.

```
N9K# copy running-config startup-config 
[########################################]  99%Configuration update aborted: timed out
N9K# 
```

This was not my first time dealing with a failed `copy run start` command on a Nexus. I've seen similar issues when the SSD begins to fail, causing the bootflash to become unresponsive or mounted in a read-only state. However, the error message is slightly different and doesn't take several minutes to time out. Furthermore, the fact that this issue happened on brand-new hardware across multiple switches made me think something else was going on aside from hardware failure.

I knew from prior experience that the "Configuration update aborted" part of this error message was generic. The "timed out" portion of the error message was the real indicator as to what's going on behind the scenes. This means that the switch was *trying* to save the configuration, but the act of saving the configuration took an excessively long time, so the switch bailed out of the operation entirely.

After gathering logs from the affected switches and doing some research, I suspected there was something within each switch's configuration that could be causing this issue. I grabbed a fresh switch in our lab with no configuration on it.

```
switch# show running-config | count
146
switch# 
```

First, I decided to benchmark how long a "normal" `copy running-config startup-config` takes by running the `show clock` command before and after saving configuration. I chained all three commands together to eliminate any human response times from the equation.

```
switch# show clock ; copy running-config startup-config ; show clock
15:50:41.484 UTC Mon Jan 03 2022
Time source is NTP
[########################################] 100%
Copy complete, now saving to disk (please wait)...
Copy complete.
15:50:46.579 UTC Mon Jan 03 2022
Time source is NTP
switch# 
```

This shows that on a fresh switch with minimal configuration, the `copy running-config startup-config` command started at *around* 15:50:41.484 switchtime and ended at *around* 15:50:46.579 switchtime. This means that it took the switch *about* five seconds to save its configuration with a default, minimum set of configuration.

Next, I applied almost all of the configuration from one of the customer's switches. I sanitized certain bits and pieces of the configuration (such as user accounts, mgmt0 interface configuration, etc.) but left mostly everything else. Sanitizing very little of the configuration is important because *if* some element of configuration is needed to reproduce the issue, then we need to leave as much of it as intact as possible. We don't currently know what specific part of the configuration is important. If we already knew this, we wouldn't need to reproduce this issue in the first place - therefore, we need to leave as much of the configuration intact as possible so that we don't accidentally ruin our chances of reproducing the issue.

Once the configuration was applied, I had just under 2,000 lines of configuration present in the running-config of the switch.

```
N9K# show running-config | count
1946
N9K# 
```

Next, I decided to benchmark a `copy running-config startup-config` with this configuration using the same tactic as before.

```
N9K# show clock ; copy running-config startup-config ; show clock
16:10:54.004 UTC Mon Jan 03 2022
Time source is NTP
[########################################]  99%Configuration update aborted: timed out
16:18:55.186 UTC Mon Jan 03 2022
Time source is NTP
N9K# 
```

As you can see, we reproduced the exact issue! The switch took about 8 minutes to try to save the configuration. After 8 minutes, the operation bailed out.

Now that we have some configuration that reproduces the issue, we can strip out bits and pieces of the configuration until we find what combination triggers the issue. After some poking and prodding, I found that the customer had an MOTD (Message of the Day) banner similar to the one shown here.

```
N9K# show running-config | begin banner | head lines 5
banner motd Q
##################################
# This is an example MOTD banner #
##################################
Q
N9K# 
```

Coincidentally, they also had the CLI aliases shown here configured. Specifically, the `q` alias (probably short for "quit") caught my eye.

```
N9K# show running-config | section alias
cli alias name wr copy running-config startup-config
cli alias name q exit
N9K# 
```

You may notice that the MOTD banner delimiter, `Q`, is identical to the character used by the CLI alias. If we remove the MOTD banner entirely, we are no longer able to reproduce the issue.

```
N9K# configure terminal
Enter configuration commands, one per line. End with CNTL/Z.
N9K(config)# no banner motd
N9K(config)# end
N9K# show clock ; copy running-config startup-config ; show clock
16:45:36.726 UTC Mon Jan 03 2022
Time source is NTP
[########################################] 100%
Copy complete, now saving to disk (please wait)...
Copy complete.
16:45:43.274 UTC Mon Jan 03 2022
Time source is NTP
N9K# 
```

Similarly, if I re-configure the MOTD banner, but remove `cli alias name q exit` from the configuration, then I am no longer able to reproduce the issue:

```
N9K# configure terminal
Enter configuration commands, one per line. End with CNTL/Z.
N9K(config)# banner motd Q
> ##################################
> # This is an example MOTD banner #
> ##################################
> Q
N9K(config)# no cli alias name q exit
N9K(config)# end
N9K# show running-config | section alias
cli alias name wr copy running-config startup-config
N9K# show clock ; copy running-config startup-config ; show clock
16:49:52.100 UTC Mon Jan 03 2022
Time source is NTP
[########################################] 100%
Copy complete, now saving to disk (please wait)...
Copy complete.
16:49:58.699 UTC Mon Jan 03 2022
Time source is NTP
N9K# 
```

Now that we know these two configuration elements are related to this issue, we can dig into the issue deeper. One thing you may wonder is, why does a normal `copy running-config startup-config` command take 5 seconds to execute? Surely it doesn't take a Nexus switch 5 seconds to copy a big string in system memory to a file on an SSD? In reality, `copy running-config startup-config` is much more complicated than just writing a string to a file. In the background, NX-OS keeps two versions of the startup-config - a "binary" version, and an "ASCII" version.

The binary version is a gunzipped archive file (.tar.gz) consisting of each individual NX-OS software component's (OSPF, BGP, EIGRP, URIB, etc.) startup-config. The file format used to store this information varies, but many software components have standardized on a feature known as PSS (Persistent Storage Service), which is a lightweight key-value pair database used to store configuration and operational state. This binary version is optimized so that the switch can quickly load the configuration after the switch is reloaded. If the binary version *didn't* exist, then when a Nexus switch reloads, it would restore the startup-config line-by-line (which could take a long time, depending on the contents and length of the configuration).

Whenever you apply a line of configuration in the CLI, that line of configuration usually gets translated to an object within the relevant NX-OS software component's PSS database. For example, if you create a new OSPF process by configuring `router ospf 1`, a new OSPF process will be spun up, and information about that process's configuration (such as the router ID, interfaces activated under the protocol, etc.) will be stored within a PSS database created by the process. The OSPF process does *not* retroactively parse the running-config or startup-config to figure out how it should be configured.

An example of where the binary startup-config is located and how to extract it is shown here.

```
switch# configure terminal
Enter configuration commands, one per line. End with CNTL/Z.
switch(config)# feature bash-shell
switch(config)# end
switch# run bash sudo su -
root@switch#mkdir /bootflash/binary_startup_configuration/
root@switch#tar xvf /mnt/cfg/0/bin/bin_cfg.tar.gz -C /bootflash/binary_startup_configuration/
./
./cmos-snapshot
./knownhosts.tar
./copp_startup_cfg_user.trm
./copp_startup_cfg_user.dat
./copp_startup_cfg_user.map
./copp_startup_cfg_user.pss
./copp_startup_cfg_node.trm
./copp_startup_cfg_node.dat
./copp_startup_cfg_node.map
./copp_startup_cfg_node.pss
./copp_startup_cfg_ctrl.trm
./copp_startup_cfg_ctrl.dat
./copp_startup_cfg_ctrl.map
./copp_startup_cfg_ctrl.pss
./snmpd_startup_cfg.trm
./snmpd_startup_cfg.dat
./snmpd_startup_cfg.map
./snmpd_startup_cfg.pss
./snmpd_def_startup_cfg.trm
./snmpd_def_startup_cfg.dat
./snmpd_def_startup_cfg.map
./snmpd_def_startup_cfg.pss
./new-ipconf-ver
./aclmgr_start_cfg_user.trm
./aclmgr_start_cfg_user.dat
./aclmgr_start_cfg_user.map
./aclmgr_start_cfg_user.pss
./aclmgr_start_cfg_node.trm
./aclmgr_start_cfg_node.dat
```

> **Note**: It goes without saying that messing with the contents of these files is not supported by Cisco.

The ASCII version is a plaintext file consisting of each individual line of configuration. This is the output of `show startup-config` that we are used to in NX-OS. An example of where the ASCII startup-config is located is shown here.

```
switch# configure terminal
Enter configuration commands, one per line. End with CNTL/Z.
switch(config)# feature bash-shell
switch(config)# end
switch# run bash sudo su -
root@switch#cat /var/sysmgr/startup-cfg/ascii/system.cfg | more

!Command: show startup-config
!Time: Fri Dec 10 00:16:48 2021
!Startup config saved at: Fri Dec 10 00:16:48 2021


version 9.3(8) Bios:version 05.45 
vdc switch id 1
  limit-resource vlan minimum 16 maximum 4094
  limit-resource vrf minimum 2 maximum 4096
  limit-resource port-channel minimum 0 maximum 511
  limit-resource u4route-mem minimum 248 maximum 248
  limit-resource u6route-mem minimum 96 maximum 96
  limit-resource m4route-mem minimum 58 maximum 58
  limit-resource m6route-mem minimum 8 maximum 8
no password strength-check
```

> **Note**: Of course, you could just run `show startup-config` to display this same file - the two are identical.

In NX-OS, when you execute the `copy running-config startup-config` command, NX-OS compiles the binary version of the startup-config and generates the ASCII running-config. After generating this ASCII running-config, NX-OS's MDP (Model Driven Programmability) set of features validates relevant parts of the ASCII startup-config can be translated to an object model through a component named DME (Data Management Engine). DME is essentially a single large data structure that houses the configuration and operational status of multiple NX-OS software components. This data structure can be modified through the CLI when you make configuration changes like normal, or it can be fetched or modified using NX-API, or you can stream the values of specific keys in the data structure through telemetry.

A visual example of how DME "sits" in between each software component's object store (PSS), the CLI, and other various configuration methods is shown in the excerpt below from [Mike Wiebe's BRKDCN-2025 2020 Cisco Live session](https://www.ciscolive.com/on-demand/on-demand-library.html?search=BRKDCN-2025&search=BRKDCN-2025%2C+BRKDCN-2025#/session/16360600455420017FSx).

![]({{ site.baseurl }}/images/2022/tac-tales-saving-configuration-is-for-quitters/dme.png)

Here is where we'll bring everything back to the issue this customer faced. A log showing how MDP processes the ASCII startup-config is shown below.

```
N9K# configure terminal
N9K(config)# feature bash-shell
N9K(config)# end
N9K# run bash sudo su -
root@N9K#cat /tmp/system.mdp_save.cfg | more
`config terminal`
save non dme command
mode 1,  cli: config terminal
 `version 9.3(3) Bios:version 07.69 `
save non dme command
mode 0,  cli: version 9.3(3) Bios:version 07.69 
 `hostname N9K`
skip dme command
mode 0,  cli: hostname N9K
 `install feature-set mpls`
skip dme command
mode 0,  cli: install feature-set mpls
```

As you can see, this log shows how MDP is processing the ASCII configuration line-by-line and identifying lines of configuration that should be present in the DME data structure. If we scroll further down in the log, we see how the MOTD banner configuration is processed.

```
 `banner motd Q`
skip dme command
mode 0,  cli: banner motd Q
 `exit`
save non dme command
mode 2,  cli: exit
 `no ip domain-lookup`
Syntax error while parsing 'no ip domain-lookup'

`copp profile strict`
Syntax error while parsing 'copp profile strict'
```

As you can see, the command immediately after the `banner motd Q` command is incorrectly translated to the `exit` command because of the `cli alias name q exit` configuration command. This causes the CLI parser to exit the global configuration context, causing all subsequent commands in the ASCII startup-config to fail because they are executed in the wrong context. These subsequent syntax errors cause MDP to take an exceptionally long time to process the rest of the ASCII startup-config, eventually causing the entire `copy running-config startup-config` command to time out entirely.

This bug was tracked via [CSCvs40645](https://bst.cloudapps.cisco.com/bugsearch/bug/CSCvs40645) and is fixed in NX-OS software releases 9.3(4) and later.

My personal opinion? The moral of this story is that **software development is hard**. No amount of testing or Q/A efforts will fully prepare your software project for the unique ways users choose to use your code. There is no such thing as bug-free code; to write code is to create bugs!
