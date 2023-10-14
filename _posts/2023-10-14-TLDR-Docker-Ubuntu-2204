---
layout: post
title: "TL;DR: Install Docker on Ubuntu 22.04"
---

This post serves as a quick reference for installing Docker on Ubuntu 22.04. It is not a wholistic replacement for the [official Docker documentation on this subject](https://docs.docker.com/engine/install/ubuntu/), but serves as a quick summary of the steps.

## TL;DR

In a hurry? Execute the below commands. Simply copy and paste them into your terminal window. Note that you will be prompted for your password at the beginning of the process.

```bash
sudo -v
sudo apt-get -y install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" |  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get -y update
sudo apt-get -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
exec su -l $USER
docker --version
docker run hello-world
```

Docker should now be installed on your Ubuntu 22.04 system.

## Docker Installation Step-by-Step

If you'd like a *bit* more information about what each command does, follow the steps below.

### Confirm Ubuntu Version

Use the `cat /etc/os-release` command to confirm you are current running Ubuntu 22.04.

```bash
cat /etc/os-release
```

Example output of this command is below:

```console
christopher@ubuntu:~$ cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04.3 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.3 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian
HOME_URL="https://www.ubuntu.com/"
SUPPORT_URL="https://help.ubuntu.com/"
BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
UBUNTU_CODENAME=jammy
```

### Add Docker GPG Key

Add the Docker GPG key to the apt package manager's keyring. This ensure the Docker repository we will add next is trusted.

```bash
sudo apt-get -y install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Example output of these commands is below:

```console
christopher@awx:~$ sudo apt-get -y install ca-certificates curl gnupg
[sudo] password for christopher:
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
ca-certificates is already the newest version (20230311ubuntu0.22.04.1).
ca-certificates set to manually installed.
curl is already the newest version (7.81.0-1ubuntu1.14).
curl set to manually installed.
gnupg is already the newest version (2.2.27-3ubuntu2.1).
gnupg set to manually installed.
The following packages were automatically installed and are no longer required:
  libflashrom1 libftdi1-2
Use 'sudo apt autoremove' to remove them.
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.

christopher@awx:~$ sudo install -m 0755 -d /etc/apt/keyrings
christopher@awx:~$
christopher@awx:~$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
christopher@awx:~$
christopher@awx:~$ sudo chmod a+r /etc/apt/keyrings/docker.gpg
christopher@awx:~$
```

### Add apt Repository and Update Package Cache

Add the Docker repository to the apt package manager's list of repositories and update the package cache.

```bash
echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" |  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get -y update
```

Example output of these commands is below:

```console
christopher@awx:~$ echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

christopher@awx:~$ sudo apt-get -y update
Hit:1 http://us.archive.ubuntu.com/ubuntu jammy InRelease
Get:2 https://download.docker.com/linux/ubuntu jammy InRelease [48.9 kB]
Get:3 http://us.archive.ubuntu.com/ubuntu jammy-updates InRelease [119 kB]
Get:4 https://download.docker.com/linux/ubuntu jammy/stable amd64 Packages [22.2 kB]
Get:5 http://us.archive.ubuntu.com/ubuntu jammy-backports InRelease [109 kB]
Get:6 http://us.archive.ubuntu.com/ubuntu jammy-security InRelease [110 kB]
Fetched 409 kB in 0s (1,040 kB/s)
Reading package lists... Done
christopher@awx:~$
```

### Install Docker Packages

Install the Docker packages using the apt package manager.

```bash
sudo apt-get -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Example output of this command is below:

```console
christopher@awx:~$ sudo apt-get -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following packages were automatically installed and are no longer required:
  libflashrom1 libftdi1-2
Use 'sudo apt autoremove' to remove them.
The following additional packages will be installed:
  docker-ce-rootless-extras libltdl7 libslirp0 pigz slirp4netns
Suggested packages:
  aufs-tools cgroupfs-mount | cgroup-lite
The following NEW packages will be installed:
  containerd.io docker-buildx-plugin docker-ce docker-ce-cli docker-ce-rootless-extras docker-compose-plugin libltdl7
  libslirp0 pigz slirp4netns
0 upgraded, 10 newly installed, 0 to remove and 0 not upgraded.
Need to get 114 MB of archives.
After this operation, 409 MB of additional disk space will be used.
Get:1 http://us.archive.ubuntu.com/ubuntu jammy/universe amd64 pigz amd64 2.6-1 [63.6 kB]
Get:2 https://download.docker.com/linux/ubuntu jammy/stable amd64 containerd.io amd64 1.6.24-1 [28.6 MB]
Get:3 http://us.archive.ubuntu.com/ubuntu jammy/main amd64 libltdl7 amd64 2.4.6-15build2 [39.6 kB]
Get:4 http://us.archive.ubuntu.com/ubuntu jammy/main amd64 libslirp0 amd64 4.6.1-1build1 [61.5 kB]
Get:5 http://us.archive.ubuntu.com/ubuntu jammy/universe amd64 slirp4netns amd64 1.0.1-2 [28.2 kB]
Get:6 https://download.docker.com/linux/ubuntu jammy/stable amd64 docker-buildx-plugin amd64 0.11.2-1~ubuntu.22.04~jammy [28.2 MB]
Get:7 https://download.docker.com/linux/ubuntu jammy/stable amd64 docker-ce-cli amd64 5:24.0.6-1~ubuntu.22.04~jammy [13.3 MB]
Get:8 https://download.docker.com/linux/ubuntu jammy/stable amd64 docker-ce amd64 5:24.0.6-1~ubuntu.22.04~jammy [22.6 MB]
Get:9 https://download.docker.com/linux/ubuntu jammy/stable amd64 docker-ce-rootless-extras amd64 5:24.0.6-1~ubuntu.22.04~jammy [9,032 kB]
Get:10 https://download.docker.com/linux/ubuntu jammy/stable amd64 docker-compose-plugin amd64 2.21.0-1~ubuntu.22.04~jammy [11.9 MB]
Fetched 114 MB in 1s (98.5 MB/s)
Selecting previously unselected package pigz.
(Reading database ... 74209 files and directories currently installed.)
Preparing to unpack .../0-pigz_2.6-1_amd64.deb ...
Unpacking pigz (2.6-1) ...
Selecting previously unselected package containerd.io.
Preparing to unpack .../1-containerd.io_1.6.24-1_amd64.deb ...
Unpacking containerd.io (1.6.24-1) ...
Selecting previously unselected package docker-buildx-plugin.
Preparing to unpack .../2-docker-buildx-plugin_0.11.2-1~ubuntu.22.04~jammy_amd64.deb ...
Unpacking docker-buildx-plugin (0.11.2-1~ubuntu.22.04~jammy) ...
Selecting previously unselected package docker-ce-cli.
Preparing to unpack .../3-docker-ce-cli_5%3a24.0.6-1~ubuntu.22.04~jammy_amd64.deb ...
Unpacking docker-ce-cli (5:24.0.6-1~ubuntu.22.04~jammy) ...
Selecting previously unselected package docker-ce.
Preparing to unpack .../4-docker-ce_5%3a24.0.6-1~ubuntu.22.04~jammy_amd64.deb ...
Unpacking docker-ce (5:24.0.6-1~ubuntu.22.04~jammy) ...
Selecting previously unselected package docker-ce-rootless-extras.
Preparing to unpack .../5-docker-ce-rootless-extras_5%3a24.0.6-1~ubuntu.22.04~jammy_amd64.deb ...
Unpacking docker-ce-rootless-extras (5:24.0.6-1~ubuntu.22.04~jammy) ...
Selecting previously unselected package docker-compose-plugin.
Preparing to unpack .../6-docker-compose-plugin_2.21.0-1~ubuntu.22.04~jammy_amd64.deb ...
Unpacking docker-compose-plugin (2.21.0-1~ubuntu.22.04~jammy) ...
Selecting previously unselected package libltdl7:amd64.
Preparing to unpack .../7-libltdl7_2.4.6-15build2_amd64.deb ...
Unpacking libltdl7:amd64 (2.4.6-15build2) ...
Selecting previously unselected package libslirp0:amd64.
Preparing to unpack .../8-libslirp0_4.6.1-1build1_amd64.deb ...
Unpacking libslirp0:amd64 (4.6.1-1build1) ...
Selecting previously unselected package slirp4netns.
Preparing to unpack .../9-slirp4netns_1.0.1-2_amd64.deb ...
Unpacking slirp4netns (1.0.1-2) ...
Setting up docker-buildx-plugin (0.11.2-1~ubuntu.22.04~jammy) ...
Setting up containerd.io (1.6.24-1) ...
Created symlink /etc/systemd/system/multi-user.target.wants/containerd.service → /lib/systemd/system/containerd.service.
Setting up docker-compose-plugin (2.21.0-1~ubuntu.22.04~jammy) ...
Setting up libltdl7:amd64 (2.4.6-15build2) ...
Setting up docker-ce-cli (5:24.0.6-1~ubuntu.22.04~jammy) ...
Setting up libslirp0:amd64 (4.6.1-1build1) ...
Setting up pigz (2.6-1) ...
Setting up docker-ce-rootless-extras (5:24.0.6-1~ubuntu.22.04~jammy) ...
Setting up slirp4netns (1.0.1-2) ...
Setting up docker-ce (5:24.0.6-1~ubuntu.22.04~jammy) ...
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /lib/systemd/system/docker.service.
Created symlink /etc/systemd/system/sockets.target.wants/docker.socket → /lib/systemd/system/docker.socket.
Processing triggers for man-db (2.10.2-1) ...
Processing triggers for libc-bin (2.35-0ubuntu3.4) ...
Scanning processes...
Scanning linux images...

Running kernel seems to be up-to-date.

No services need to be restarted.

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.
christopher@awx:~$
```

### Add Current User to Docker Group and Re-Log

Add your current user account to the `docker` group, which will enable you to execute `docker` commands without escalating privileges through `sudo`. Then, start a new shell session with the current user account to ensure the group membership takes effect.

```bash
sudo usermod -aG docker $USER
exec su -l $USER
```

Example output of this command is below:

```console
christopher@awx:~$ sudo usermod -aG docker $USER
christopher@awx:~$
christopher@awx:~$ exec su -l $USER
Password:
christopher@awx:~$
```

### Confirm Docker Version and Functionality

User the `docker --version` command to confirm Docker is installed and that your user account can execute `docker` commands. Then, run a new container from the `hello-world` image to confirm Docker is functioning properly.

```bash
docker --version
docker run hello-world
```

Example output of these commands is below:

```console
christopher@awx:~$ docker --version
Docker version 24.0.6, build ed223bc

christopher@awx:~$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
719385e32844: Pull complete
Digest: sha256:88ec0acaa3ec199d3b7eaf73588f4518c25f9d34f58ce9a0df68429c5af48e8d
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

Docker should now be installed on your Ubuntu 22.04 system.
