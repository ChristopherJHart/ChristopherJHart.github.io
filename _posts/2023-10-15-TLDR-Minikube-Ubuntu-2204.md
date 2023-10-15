---
layout: post
title: "TL;DR: Install Minikube on Ubuntu 22.04"
---

This post serves as a quick reference for installing Minikube on Ubuntu 22.04. It is not a wholistic replacement for the [official Minikube documentation on this subject](https://minikube.sigs.k8s.io/docs/start/), but serves as a quick summary of the steps.

## Prerequisites

This post assumes that you already have Docker installed on your Ubuntu 22.04 system. If this is not the case, reference the ["TL;DR: Install Docker on Ubuntu 22.04" post]({{ site.baseurl }}/TLDR-Docker-Ubuntu-2204/).

## TL;DR

In a hurry? Execute the below commands. Simply copy and paste them into your terminal window. Note that you will be prompted for your password at the beginning of the process.

```bash
sudo -v
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
minikube start --cpus=4 --memory=6g --addons=ingress
alias kubectl="minikube kubectl --"
```

Minikube should now be installed on your Ubuntu 22.04 system, and a cluster with access to 4 CPUs and 6 gigabytes of memory should be running.

## Minikube Installation Step-by-Step

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

### Install Minikube Binary

Download the Minikube binary and install it into a filepath within your `$PATH` environment variable. `/usr/local/bin` is usually an acceptable place to install binaries.

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

Example output of these commands is below:

```console
christopher@awx:~$ curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 82.4M  100 82.4M    0     0  68.5M      0  0:00:01  0:00:01 --:--:-- 68.5M
christopher@awx:~$ sudo install minikube-linux-amd64 /usr/local/bin/minikube
[sudo] password for christopher:
christopher@awx:~$
```

### Confirm Minikube Installation

Execute the `minikube version` command to confirm that the Minikube binary is successfully installed.

```bash
minikube version
```

Example output of this command is below:

```console
christopher@awx:~$ minikube version
minikube version: v1.31.2
commit: fd7ecd9c4599bef9f04c0986c4a0187f98a4396e
```

### Start Minikube Cluster

Use the `minikube start` command to start a Minikube cluster. The below example starts a cluster with access to 4 CPUs and 6 gigabytes of memory. It also enables the `ingress` addon, which allows us to access services deployed within the Minikube cluster.

```bash
minikube start --cpus=4 --memory=6g --addons=ingress
```

Example output of this command is below:

```console
christopher@awx:~$ minikube start --cpus=4 --memory=6g --addons=ingress
😄  minikube v1.31.2 on Ubuntu 22.04 (kvm/amd64)
✨  Automatically selected the docker driver. Other choices: none, ssh
📌  Using Docker driver with root privileges
👍  Starting control plane node minikube in cluster minikube
🚜  Pulling base image ...
💾  Downloading Kubernetes v1.27.4 preload ...
    > preloaded-images-k8s-v18-v1...:  393.21 MiB / 393.21 MiB  100.00% 71.17 M
    > gcr.io/k8s-minikube/kicbase...:  447.61 MiB / 447.62 MiB  100.00% 57.34 M
🔥  Creating docker container (CPUs=4, Memory=6144MB) ...
🐳  Preparing Kubernetes v1.27.4 on Docker 24.0.4 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔗  Configuring bridge CNI (Container Networking Interface) ...
🔎  Verifying Kubernetes components...
    ▪ Using image registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20230407
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
    ▪ Using image registry.k8s.io/ingress-nginx/controller:v1.8.1
    ▪ Using image registry.k8s.io/ingress-nginx/kube-webhook-certgen:v20230407
🔎  Verifying ingress addon...
🌟  Enabled addons: storage-provisioner, default-storageclass, ingress
💡  kubectl not found. If you need it, try: 'minikube kubectl -- get pods -A'
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

### Add kubectl Alias

Set up an alias for your current shell session that allows you to execute `kubectl` commands against the Minikube cluster without needing to prepend the `minikube` command.

```bash
alias kubectl="minikube kubectl --"
```

Note that this command will need to be executed on each unique shell session to the system. If you'd like to avoid this, add this command to your shell's configuration file (e.g. `~/.bashrc`, `~/.bash_profile`, `~/.profile`, etc.).

Example output of this command is below:

```console
christopher@awx:~$ kubectl version --short
Command 'kubectl' not found, but can be installed with:
sudo snap install kubectl
christopher@awx:~$
christopher@awx:~$ alias kubectl="minikube kubectl --"
christopher@awx:~$
christopher@awx:~$ kubectl version --short
Flag --short has been deprecated, and will be removed in the future. The --short output will become the default.
Client Version: v1.27.4
Kustomize Version: v5.0.1
Server Version: v1.27.4
```

Minikube should now be installed on your Ubuntu 22.04 system, and a cluster with access to 4 CPUs and 6 gigabytes of memory should now be running.
