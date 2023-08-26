---
layout: post
title: Cisco ASR 1000 Routers - Booting Cisco IOS from TFTP in ROMMON Mode
---

While working with Cisco ASR 1000 series routers, you may find yourself in a situation where the router (or a Route Processor, in the case of modular routers) will only boot into ROMMON mode:

```
Initializing Hardware ...

Calculating the ROMMON CRC ... CRC is correct


System Bootstrap, Version 16.9(5r), RELEASE SOFTWARE
Copyright (c) 1994-2019  by cisco Systems, Inc.

Current image running: Boot ROM1
Last reset cause: LocalSoft

ASR1000-RP2 platform with 8388608 Kbytes of main memory

rommon 1 >
```

If there are no valid images present on the bootflash or an attached hard disk drive, and you do not have a USB flash drive with a valid image, you will need to boot the router into Cisco IOS from ROMMON mode using a TFTP server. This is a fairly simple process, but it is not well documented. This post will walk you through the process.

## Prerequisites

There are three prerequisites needed to execute this process:

1. The management interface (GigabitEthernet0) of the router or Route Processor must be connected to a network where a TFTP server is accessible.
2. The TFTP server must have a valid Cisco IOS image for the router or Route Processor present.
3. You must have a connection to the console port of the router or Route Processor, either through a direct connection from an endpoint (such as a laptop) or through a [serial terminal server](https://www.cisco.com/c/en/us/support/docs/dial-access/asynchronous-connections/5466-comm-server.html).

## Summarized ROMMON TFTP Boot Procedure

In a hurry? Set the following ROMMON variables:

```
IP_ADDRESS=192.0.2.100
IP_SUBNET_MASK=255.255.255.0
DEFAULT_GATEWAY=192.0.2.1
TFTP_SERVER=192.0.2.250
TFTP_FILE=asr1000rpx86-universalk9.17.06.04.SPA.bin
```

Then, use the `boot tftp:` command to boot the router or Route Processor into Cisco IOS via TFTP.

```
rommon 8 > boot tftp:

          IP_ADDRESS: 192.0.2.100
      IP_SUBNET_MASK: 255.255.255.0
     DEFAULT_GATEWAY: 192.0.2.1
         TFTP_SERVER: 192.0.2.250
           TFTP_FILE: asr1000rpx86-universalk9.17.06.04.SPA.bin
        TFTP_MACADDR: e0:5f:b9:75:25:ff
        TFTP_VERBOSE: Progress
    TFTP_RETRY_COUNT: 18
        TFTP_TIMEOUT: 7200
        TFTP_BLKSIZE: 1460
       TFTP_CHECKSUM: Yes
          ETHER_PORT: 3
    ETHER_SPEED_MODE: Auto Detect
link up
Receiving asr1000rpx86-universalk9.17.06.04.SPA.bin from 192.0.2.250
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!File reception completed.
<snip>
Press RETURN to get started! 
```

Once the router is booted into TFTP, make sure that you:

1. Copy the Cisco IOS image from the TFTP server to the bootflash or hard disk drive of the router or Route Processor
2. Set the router's boot statement to the desired IOS image
3. Validate the configuration register has a value of `0x2102`

Otherwise, the router will most likely boot into ROMMON mode again the next time it is restarted.

## Detailed ROMMON TFTP Boot Procedure

First, view the current ROMMON variables by executing the `set` ROMMON command.

```
rommon 1 > set
PS1=rommon ! > 
BOOT_PARAM=root=/dev/ram rw console=ttyS1,9600 max_loop=64
NT_K=0:0:0:0
PLATFORM_MAX_INTERFACES=
MCP_STARTUP_TRACEFLAGS=00000000:00000000
?=1
CONFIG_FILE=
BOOTLDR=
DEVICE_MANAGED_MODE=autonomous
LICENSE_ACTIVE_LEVEL=adventerprise,all:asr1001;
LICENSE_SUITE=
LICENSE_BOOT_LEVEL=adventerprise,all:asr1001;
BSI=0
RANDOM_NUM=1467669512
RET_2_RTS=17:37:43 UTC Sat Aug 26 2023
RET_2_RCALTS=1693071463
```

Next, set the `IP_ADDRESS` ROMMON variable to the IPv4 address that should be assigned to the management interface (GigabitEthernet0) of the router or Route Processor.

```
rommon 2 > IP_ADDRESS=192.0.2.100
```

Then, set the `IP_SUBNET_MASK` ROMMON variable to the subnet mask that should be assigned to the management interface (GigabitEthernet0) of the router or Route Processor.

```
rommon 3 > IP_SUBNET_MASK=255.255.255.0
```

Set the `DEFAULT_GATEWAY` ROMMON variable to the default gateway that should be assigned to the management interface (GigabitEthernet0) of the router or Route Processor.

```
rommon 4 > DEFAULT_GATEWAY=192.0.2.1
```

Set the `TFTP_SERVER` ROMMON variable to the IPv4 address of the TFTP server.

```
rommon 5 > TFTP_SERVER=192.0.2.250
```

> **Note**: In this example, the TFTP server is located in the same 192.0.2.0/24 subnet as the IP address assigned to the management interface of the router/Route Processor. However, the TFTP server can be located in a different subnet, as long as the router/Route Processor has the `DEFAULT_GATEWAY` ROMMON variable pointing to an IPv4 address of a router that can route to the TFTP server.

Set the `TFTP_FILE` ROMMON variable to the filename of the Cisco IOS image that should be downloaded from the TFTP server.

```
rommon 6 > TFTP_FILE=asr1000rpx86-universalk9.17.06.04.SPA.bin
```

View the modified ROMMON variables by executing the `set` ROMMON command. Validate that all ROMMON variables were set correctly.

```
rommon 7 > set
PS1=rommon ! > 
BOOT_PARAM=root=/dev/ram rw console=ttyS1,9600 max_loop=64
NT_K=0:0:0:0
PLATFORM_MAX_INTERFACES=
MCP_STARTUP_TRACEFLAGS=00000000:00000000
CONFIG_FILE=
BOOTLDR=
DEVICE_MANAGED_MODE=autonomous
LICENSE_ACTIVE_LEVEL=adventerprise,all:asr1001;
LICENSE_SUITE=
BOOT=harddisk:asr1000rpx86-universalk9.17.06.04.SPA.bin,12;
LICENSE_BOOT_LEVEL=adventerprise,all:asr1001;
BSI=0
RANDOM_NUM=1467669512
RET_2_RTS=17:37:43 UTC Sat Aug 26 2023
RET_2_RCALTS=1693071463
?=0
IP_ADDRESS=192.0.2.100
IP_SUBNET_MASK=255.255.255.0
DEFAULT_GATEWAY=192.0.2.1
TFTP_SERVER=192.0.2.250
TFTP_FILE=asr1000rpx86-universalk9.17.06.04.SPA.bin
```

Lastly, execute the `boot tftp:` ROMMON command to boot the router or Route Processor into Cisco IOS via TFTP using the ROMMON variables we've defined.

```
rommon 8 > boot tftp:

          IP_ADDRESS: 192.0.2.100
      IP_SUBNET_MASK: 255.255.255.0
     DEFAULT_GATEWAY: 192.0.2.1
         TFTP_SERVER: 192.0.2.250
           TFTP_FILE: asr1000rpx86-universalk9.17.06.04.SPA.bin
        TFTP_MACADDR: e0:5f:b9:75:25:ff
        TFTP_VERBOSE: Progress
    TFTP_RETRY_COUNT: 18
        TFTP_TIMEOUT: 7200
        TFTP_BLKSIZE: 1460
       TFTP_CHECKSUM: Yes
          ETHER_PORT: 3
    ETHER_SPEED_MODE: Auto Detect
link up
Receiving asr1000rpx86-universalk9.17.06.04.SPA.bin from 192.0.2.250
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!File reception completed.
Boot image size = 1230723713 (0x495b5a81) bytes

Package header rev 1 structure detected
Calculating SHA-1 hash...done
validate_package_cs: SHA-1 hash:
        calculated be4a195e:4e48e83d:7e7d4687:6df03c7e:e95c7294
        expected   be4a195e:4e48e83d:7e7d4687:6df03c7e:e95c7294
Validating main package signatures

RSA Signed RELEASE Image Signature Verification Successful.
Image validated
Warning: ignoring ROMMON var "BOOT_PARAM"
Warning: ignoring ROMMON var "BOOT_PARAM" 
Aug 26 17:49:06.471: %BOOT-5-OPMODE_LOG: R0/0: binos: System booted in AUTONOMOUS mode

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



Cisco IOS Software [Bengaluru], ASR1000 Software (X86_64_LINUX_IOSD-UNIVERSALK9-M), Version 17.6.4, RELEASE SOFTWARE (fc1)
Technical Support: http://www.cisco.com/techsupport
Copyright (c) 1986-2022 by Cisco Systems, Inc.
Compiled Sun 14-Aug-22 08:52 by mcpre


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



All TCP AO KDF Tests Pass
cisco ASR1006 (RP2) processor (revision RP2) with 4151787K/6147K bytes of memory.
Processor board ID FOXREDACTED
Router operating mode: Autonomous
32768K bytes of non-volatile configuration memory.
8388608K bytes of physical memory.
1933311K bytes of eUSB flash at bootflash:.
78085207K bytes of SATA hard disk at harddisk:.

Press RETURN to get started! 
```

It is important to note that copying a Cisco IOS image from a TFTP server from ROMMON mode does ***not*** copy the Cisco IOS image to the bootflash or hard disk drive file systems of the router or Route Processor. This needs to be done manually after the router/Route Processor has booted from ROMMON into Cisco IOS via TFTP. Otherwise, the router will most likely boot into ROMMON mode again the next time it is restarted.

```
Router#dir bootflash: | include bin
Router#
Router#dir harddisk: | include bin
Router#
Router#copy tftp://192.0.2.250/asr1000rpx86-universalk9.17.06.04.SPA.bin bootflash:
Accessing tftp://192.0.2.250/asr1000rpx86-universalk9.17.06.04.SPA.bin...
Loading asr1000rpx86-universalk9.17.06.04.SPA.bin from 192.0.2.250 (via GigabitEthernet0): !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
[OK - 1230723713 bytes]

1230723713 bytes copied in 940.532 secs (1308540 bytes/sec)
Router#                                                                          
Router#dir bootflash: | include bin                                                    
19      -rw-       1230723713  Aug 26 2023 18:29:48 +00:00  asr1000rpx86-universalk9.17.06.04.SPA.bin
Router# 
```

Next, ensure the router's boot statement is configured to point to the desired Cisco IOS image. An example of how to change the boot statement is shown below.

```
Router#show boot
BOOT variable = harddisk:asr1000rpx86-universalk9.17.06.04.SPA.bin,12;
CONFIG_FILE variable = 
BOOTLDR variable does not exist
Configuration register is 0x2100

Standby BOOT variable = harddisk:asr1000rpx86-universalk9.17.06.04.SPA.bin,12;
Standby CONFIG_FILE variable = 
Standby BOOTLDR variable does not exist
Standby Configuration register is 0x2100 

Router#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Router(config)#no boot system
Router(config)#boot system bootflash:asr1000rpx86-universalk9.17.06.04.SPA.bin
Router(config)#end
Router#copy running-config startup-config
Building configuration...
[OK]
```

Finally, validate the configuration register is set to a value of `0x2102`. This will ensure the router or Route Processor will boot into the desired Cisco IOS image, and the saved startup configuration is properly loaded when the router is restarted.

```
Router#show boot
BOOT variable = bootflash:asr1000rpx86-universalk9.17.06.04.SPA.bin,12;
CONFIG_FILE variable = 
BOOTLDR variable does not exist
Configuration register is 0x2100

Standby BOOT variable = bootflash:asr1000rpx86-universalk9.17.06.04.SPA.bin,12;
Standby CONFIG_FILE variable = 
Standby BOOTLDR variable does not exist
Standby Configuration register is 0x2100

Router#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Router(config)#config-register 0x2102 
Router(config)#end
Router#show boot
BOOT variable = bootflash:asr1000rpx86-universalk9.17.06.04.SPA.bin,12;
CONFIG_FILE variable = 
BOOTLDR variable does not exist
Configuration register is 0x2102

Standby BOOT variable = bootflash:asr1000rpx86-universalk9.17.06.04.SPA.bin,12;
Standby CONFIG_FILE variable = 
Standby BOOTLDR variable does not exist
Standby Configuration register is 0x2102
```

## References

* ["Troubleshooting the Upgrade" section of the "Troubleshooting Initial Startup Problems" chapter in the Cisco ASR 1000 Series Router Hardware Installation Guide document](https://www.cisco.com/c/en/us/td/docs/routers/asr1000/install/guide/asr1routers/asr-1000-series-hig/asr-hig-tbl.html#con_1043889).
