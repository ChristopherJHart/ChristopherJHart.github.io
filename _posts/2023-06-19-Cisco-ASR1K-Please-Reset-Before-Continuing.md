---
layout: post
title: Cisco ASR 1000 Routers - 'Please reset before continuing' or 'Please reset before booting' in ROMMON
---

While working in ROMMON mode of Cisco's ASR 1000 series of routers, you may encounter the `Please reset before continuing` or `Please reset before booting` error messages that prevent most commands from being executed after sending a break signal through the console port of the router during boot-up. An example of these error messages is shown below:

```
Router#reload
Proceed with reload? [confirm]

*Jun 17 21:20:26.439: %SYS-5-RELOAD: Reload requested by console. Reload Reason: Reload Command.Jun 17 21:20:32.525: %PMAN-5-EXITACTION: R0/0: pvp: Process manager is exiting: process exit with reload chassis code

Initializing Hardware ...

System integrity status: 00000610

System Bootstrap, Version 17.3(1r), RELEASE SOFTWARE
Copyright (c) 1994-2020  by cisco Systems, Inc.

Current image running: Boot ROM0
Last reset cause: LocalSoft

ASR1001-X platform with 8388608 Kbytes of main memory


autoboot: aborted due to user interrupt
rommon 1 >
rommon 2 >dir bootflash:
Please reset before continuing
rommon 3 >boot 
Please reset before booting
```

This can be frustrating to work with if you need to browse the router's file system to find the appropriate Cisco IOS binary image file you want to boot from. You've entered a ["chicken-and-egg" problem](https://en.wikipedia.org/wiki/Chicken_or_the_egg) - in order to boot from the correct image, you need to enter ROMMON using a break signal, but by entering a break signal, you prevent yourself from entering many commands in ROMMON.

The root cause of this issue is software defect [CSCtq26164](https://bst.cloudapps.cisco.com/bugsearch/bug/CSCtq26164). We can work around this issue by following the workaround detailed within the software defect.

First, we will disable the "autoboot" functionality that allows the router to automatically boot from a Cisco IOS binary image file through the `confreg 0x2100` ROMMON command. This effectively forces the router to boot into ROMMON every time it is reloaded by clearing the [first three bits of the configuration register](https://www.cisco.com/c/en/us/support/docs/routers/10000-series-routers/50421-config-register-use.html), which control the boot behavior of the router. An example of this command is shown below.

```
rommon 4 >confreg 0x2100
```

Next, we will issue the `reset` command to reboot the router. The router will automatically boot back into ROMMON, as shown below.

```
rommon 5 >reset

Resetting .......

Initializing Hardware ...

System integrity status: 00000610

System Bootstrap, Version 17.3(1r), RELEASE SOFTWARE
Copyright (c) 1994-2020  by cisco Systems, Inc.

Current image running: Boot ROM0
Last reset cause: LocalSoft

ASR1001-X platform with 8388608 Kbytes of main memory

rommon 1 >
```

Then, we will re-enable the autoboot functionality of the router by restoring the default configuration register value of 0x2102 through the `confreg 0x2102` ROMMON command.

```
rommon 1 >confreg 0x2102

You must reset or power cycle for new config to take effect
```

Now, we should be able to execute all ROMMON commands, including the `dir bootflash:` command that lets us view the contents of the file system.

```
rommon 2 >dir bootflash:
File System: EXT2/EXT3

11         16384     drwx------     lost+found
88881      4096      drwxrwxrwx     .prst_sync
363601     4096      drwxrwxrwx     .installer
22         905070380 -rw-rw-r--     asr1001x-universalk9.17.03.04a.SPA.bin
```

We can see the `asr1001x-universalk9.17.03.04a.SPA.bin` Cisco IOS binary image file is present on the `bootflash` file system. Now, let's boot the router from this file using the `boot bootflash:asr1001x-universalk9.17.03.04a.SPA.bin` ROMMON command.

```
rommon 3 >boot bootflash:asr1001x-universalk9.17.03.04a.SPA.bin
File size is 0x35f2472c
Located asr1001x-universalk9.17.03.04a.SPA.bin 
Image size 905070380 inode num 22, bks cnt 220965 blk size 8*512
#######################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################################
Boot image size = 905070380 (0x35f2472c) bytes

ROM:RSA Self Test Passed
ROM:Sha512 Self Test Passed

Package header rev 1 structure detected
Validating main package signatures

RSA Signed RELEASE Image Signature Verification Successful.
Image validated
Jun 17 21:29:40.943: %BOOT-5-OPMODE_LOG: R0/0: binos: System booted in AUTONOMOUS mode

              Restricted Rights Legend

Use, duplication, or disclosure by the Government is
subject to restrictions as set forth in subparagraph
(c) of the Commercial Computer Software - Restricted
Rights clause at FAR sec. 52.227-19 and subparagraph
(c) (1) (ii) of the Rights in Technical Data and Computer
Software clause at DFARS sec. 252.227-7013.

           Cisco Systems, Inc.
           170 West Tasman Drive
           San Jose, California 95134-1706



Cisco IOS Software [Amsterdam], ASR1000 Software (X86_64_LINUX_IOSD-UNIVERSALK9-M), Version 17.3.4a, RELEASE SOFTWARE (fc3)
Technical Support: http://www.cisco.com/techsupport
Copyright (c) 1986-2021 by Cisco Systems, Inc.
Compiled Tue 20-Jul-21 05:02 by mcpre


This software version supports only Smart Licensing as the software licensing mechanism.


PLEASE READ THE FOLLOWING TERMS CAREFULLY. INSTALLING THE LICENSE OR
LICENSE KEY PROVIDED FOR ANY CISCO SOFTWARE PRODUCT, PRODUCT FEATURE,
AND/OR SUBSEQUENTLY PROVIDED SOFTWARE FEATURES (COLLECTIVELY, THE
"SOFTWARE"), AND/OR USING SUCH SOFTWARE CONSTITUTES YOUR FULL
ACCEPTANCE OF THE FOLLOWING TERMS. YOU MUST NOT PROCEED FURTHER IF YOU
ARE NOT WILLING TO BE BOUND BY ALL THE TERMS SET FORTH HEREIN.

Your use of the Software is subject to the Cisco End User License Agreement
(EULA) and any relevant supplemental terms (SEULA) found at
http://www.cisco.com/c/en/us/about/legal/cloud-and-software/software-terms.html.

You hereby acknowledge and agree that certain Software and/or features are
licensed for a particular term, that the license to such Software and/or
features is valid only for the applicable term and that such Software and/or
features may be shut down or otherwise terminated by Cisco after expiration
of the applicable license term (e.g., 90-day trial period). Cisco reserves
the right to terminate any such Software feature electronically or by any
other means available. While Cisco may provide alerts, it is your sole
responsibility to monitor your usage of any such term Software feature to
ensure that your systems and networks are prepared for a shutdown of the
Software feature.


% Failed to initialize nvram

All TCP AO KDF Tests Pass
cisco ASR1001-X (1NG) processor (revision 1NG) with 3756727K/6147K bytes of memory.
Processor board ID FXS12345678
Router operating mode: Autonomous
6 Gigabit Ethernet interfaces
2 Ten Gigabit Ethernet interfaces
32768K bytes of non-volatile configuration memory.
8388608K bytes of physical memory.
6070271K bytes of eUSB flash at bootflash:.

No startup-config, starting autoinstall/pnp/ztp...

Autoinstall will terminate if any input is detected on console

Autoinstall trying DHCPv4 on GigabitEthernet0/0/0,GigabitEthernet0/0/1,GigabitEthernet0


         --- System Configuration Dialog ---

Would you like to enter the initial configuration dialog? [yes/no]: 
```

As you can see, the Cisco ASR 1000 router successfully booted from the Cisco IOS binary image file present on the bootflash of the switch without encountering any further error messages in ROMMON.
