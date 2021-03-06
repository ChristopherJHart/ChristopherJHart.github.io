---
layout: post
title: Cisco TCL Script Not Running Configure Replace
---

During the research phase of my networking homelab initial configuration automation article, I explored many different options on how to easily deploy a base configuration to devices that had been reset to factory defaults. One of the options I tested involved TCL scripts, where a script would reset the device to factory settings, then use the `configure replace` command to introduce a predefined configuration. This configuration would establish remote access, define IP addressing across management interfaces, create usernames and passwords, and implement any other convenient commands for the user. The `configure replace` command worked extraordinarily well: SSH could be used for remote access because cryptographic keys could be loaded into the base configuration, there was no need to reload any devices in order to revert to the base configuration, and a previous Telnet or SSH session would not be interrupted at all when the `configure replace` command is run.

I chose not to use the `configure replace` command for a number of reasons, but one of them was the fact that Cisco devices do not support the `configure replace` command when executed via the TCL shell. After many hours of testing and research, I discovered this lack of support through internal Cisco documentation. This article will illustrate this topic so that if others encounter similar issues, they will be made aware that this functionality is not supported. I will demonstrate this by showing how `configure replace` should work normally, then show what happens when executed via the TCL shell.

For this example, I have a Cisco 1841 router running IOS 15.1(4)M12A, which is the recommended software release on Cisco’s support site as of this date. A `test-config` file on the 1841’s flash contains a configuration that is automatically generated by the router, with the only difference being a change of hostname from Router to Test. As shown below, when the `configure replace flash:test-config` is run, the hostname instantly changes:

```
Router#configure replace flash:test-config
This will apply all necessary additions and deletions
to replace the current running configuration with the
contents of the specified configuration file, which is
assumed to be a complete configuration, not a partial
configuration. Enter Y if you are sure you want to proceed. ? [no]: yes
Total number of passes: 1
Rollback Done

Test#
```

Next, let’s revert back to our previous configuration and test using the TCL shell. Enter the TCL shell and attempt to run the same `configure replace` command using the exec function:

```
Router(tcl)#exec "configure replace flash:test-config"
% Configuration not allowed from TCL shell.Use 'ios_config' instead
```

This error message is a bit confusing, as the `exec` function in TCL is normally used to execute commands that are run in privileged EXEC mode of the router, while the `ios_config` function is normally used to execute commands that are run in global configuration mode. Let’s follow the TCL shell’s instructions and try to run the command using `ios_config`:

```
Router(tcl)#ios_config "configure replace flash:test-config"
Router(tcl)#
Jan 2 13:06:58.083: %SYS-5-CONFIG_I: Configured from console by vty0
Router(tcl)#
```

The command appears to have worked; we even have a syslog message suggesting that the device has been configured through a VTY line (which is used when commands are run through the TCL shell.) However, we do not see the hostname of the router change, which would have happened if the configuration was actually replaced.

When the `configure replace` command is executed in the privileged EXEC mode, it is an interactive command that requires user input and confirmation.  It could be argued that the command fails because the TCL shell is waiting for the user to type "yes" to confirm the configuration replacement. We can bypass the user input by using the `configure replace flash:test-config force` command to rule out this argument:

```
Router#configure replace flash:test-config force
Total number of passes: 1
Rollback Done

Test#
Test#configure terminal
Test(config)#hostname Router
Router(config)#end
Router#tclsh
Router(tcl)#exec "configure replace flash:test-config force"
% Configuration not allowed from TCL shell.Use 'ios_config' instead
Router(tcl)#ios_config "configure replace flash;test-config force"
Router(tcl)#
Jan 2 13:12:21.599: %SYS-5-CONFIG_I: Configured from console by vty0
Router(tcl)#
```

After doing some research internally, I found documentation that indicates using the `configure replace` command from the TCL shell (whether it be directly through the shell, or by executing a TCL script) is not a supported feature in any version of IOS. The error message that is output by the TCl shell when the `exec` function is used to execute the `configure replace` command is indicative of this, as the `configure replace` command is run in privileged EXEC mode, but the underlying function runs commands that need the permissions of global configuration mode (which can only be executed by the `ios_config` function.)