---
layout: post
title: Cisco IOS - \"No matching key exchange found\" During SSH
---

A common issue seen when attempting to SSH into a network device running Cisco's IOS or IOS-XE operating system using an OpenSSH client is as follows:

```
christopher@ubuntu-playground:~$ ssh 192.168.10.3
Unable to negotiate with 192.168.10.3 port 22: no matching key exchange method found. Their offer: diffie-hellman-group-exchange-sha1,diffie-hellman-group14-sha1,diffie-hellman-group1-sha1
```

This issue can be solved by adding the following lines to the `~/.ssh/config` file. Change the `192.168.10.3` IP address to the IP address or FQDN (Fully Qualified Domain Name) of the Cisco IOS network device:

```
Host 192.168.10.3
    KexAlgorithms +diffie-hellman-group1-sha1
```

You can also resolve this without modifying your SSH configuration file by adding `-oKexAlgorithms=+diffie-hellman-group1-sha1` to your SSH command as shown below:

```
christopher@ubuntu-playground:~$ ssh 192.168.10.3 -oKexAlgorithms=+diffie-hellman-group1-sha1
```

> **Note**: After making this change, you may receive a `no matching host key type found. Their offer: ssh-rsa` error message when attempting to SSH into the Cisco IOS network device once more. If so, add the `HostkeyAlgorithms +ssh-rsa` line underneath the relevant `Host` entry for the IP address or FQDN corresponding with the Cisco IOS network device in the `~/.ssh/config` file. Alternatively, add `-oHostkeyAlgorithms=+ssh-rsa` to your SSH command. For more information, see the [Cisco IOS - "No matching host key type found" During SSH]({{ site.baseurl }}/Cisco-IOS-SSH-Key-Type/) article.

After adding these lines to the `~/.ssh/config` file, attempt to SSH into the Cisco IOS network device once more. You should now be able to SSH into the network device successfully.

```
christopher@ubuntu-playground:~$ ssh 192.168.10.3
The authenticity of host '192.168.10.3 (192.168.10.3)' can't be established.
RSA key fingerprint is SHA256:0NLraq4nXUC/CsyVmuVCmFRwVfhSzTeYtHWT0E42L0Q.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.10.3' (RSA) to the list of known hosts.
(christopher@192.168.10.3) Password: 

C3560-CX#
```

If you add the extra parameters to your SSH command instead, you should also be able to SSH into the network device successfully.

```
christopher@ubuntu-playground:~$ ssh 192.168.10.3 -oKexAlgorithms=+diffie-hellman-group1-sha1
(christopher@192.168.10.3) Password:

C3560-CX#
```
