---
layout: post
title: "TL;DR: Install AWX with Minikube on Ubuntu 22.04"
---

This post serves as a quick reference for installing AWX through a local Minikube cluster on Ubuntu 22.04 using [AWX Operator](https://github.com/ansible/awx-operator). It is not a wholistic replacement for the [official AWX Operator documentation on this subject](https://github.com/ansible/awx-operator/blob/devel/docs/installation/basic-install.md), but serves as a quick summary of the steps.

## Prerequisites

This post assumes that you already have Docker installed and a Minikube cluster running on your Ubuntu 22.04 system. If this is not the case, reference the following two posts:

* ["TL;DR: Install Docker on Ubuntu 22.04" post]({{ site.baseurl }}/TLDR-Docker-Ubuntu-2204/)
* ["TL;DR: Install Minikube on Ubuntu 22.04" post]({{ site.baseurl }}/TLDR-Minikube-Ubuntu-2204/)

## TL;DR

In a hurry? Execute the below commands. Simply copy and paste each one into your terminal window one by one. Note that you will be prompted for your password at the beginning of the process.

```bash
mkdir awx
# Create kustomization file
cat <<EOF > awx/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=2.5.3

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.5.3

# Specify a custom namespace in which to install AWX
namespace: awx
EOF

# Apply kustomization file
kubectl apply -k awx/

# Modify kubectl namespace
kubectl config set-context --current --namespace=awx

# Wait until awx-operator pod is running and fully ready. Control+C to break out of this command.
kubectl get pods -w

# Create AWX resource file
cat <<EOF > awx/awx.yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
EOF

# Modify kustomization file to include AWX resource file
cat <<EOF > awx/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=2.5.3
  - awx.yaml

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.5.3

# Specify a custom namespace in which to install AWX
namespace: awx
EOF

# Re-apply kustomization file to pick up AWX resource file changes
kubectl apply -k awx/

# Watch installation process through logs. This may take a few minutes to complete. Wait until 
# you see the following:
#
# {"level":"info","ts":"2023-09-24T17:05:16Z","logger":"KubeAPIWarningLogger","msg":"unknown field \"status.conditions[1].ansibleResult\""}
# 
# Use Control+C to break out of this command once installation is complete.
kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager

# If you're running Minikube locally, you can access the AWX web interface by running the
# following command:
minikube service awx-service -n awx

# Confirm the IP output by the above command is reachable via curl (or simply access in your
# browser)
curl http://192.168.49.2:31227

# If you're running Minikube on a remote server, you'll need to port forward pod's web service
# port number to external IP of Minikube host. The below command exposes port 8080 on your Minikube
# cluster's external IP address. You can then access the AWX web interface by navigating to
# http://<minikube-external-ip>:8080 in your browser.
kubectl port-forward svc/awx-service --address 0.0.0.0 8080:80 &> /dev/null &

# Get the password for the AWX admin user (default username is "admin") to log into the AWX web
# interface.
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode; echo
```

You should now have a functional AWX installation running through your Minikube cluster on your Ubuntu 22.04 system.

## AWX Installation Step-by-Step

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

### Create and Apply Kustomization File

Create a directory to store AWX-related Kubernetes resource files with the `mkdir awx` command. Then, create an initial Kubernetes kustomization file for AWX.

```bash
mkdir awx
cat <<EOF > awx/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=2.5.3

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.5.3

# Specify a custom namespace in which to install AWX
namespace: awx
EOF
kubectl apply -k awx/
```

Example output of these commands is below:

```console
christopher@awx:~$ mkdir awx
christopher@awx:~$
christopher@awx:~$ cat <<EOF > awx/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=2.5.3

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.5.3

# Specify a custom namespace in which to install AWX
namespace: awx
EOF
christopher@awx:~$
christopher@awx:~$ kubectl apply -k awx/
namespace/awx created
customresourcedefinition.apiextensions.k8s.io/awxbackups.awx.ansible.com created
customresourcedefinition.apiextensions.k8s.io/awxrestores.awx.ansible.com created
customresourcedefinition.apiextensions.k8s.io/awxs.awx.ansible.com created
serviceaccount/awx-operator-controller-manager created
role.rbac.authorization.k8s.io/awx-operator-awx-manager-role created
role.rbac.authorization.k8s.io/awx-operator-leader-election-role created
clusterrole.rbac.authorization.k8s.io/awx-operator-metrics-reader created
clusterrole.rbac.authorization.k8s.io/awx-operator-proxy-role created
rolebinding.rbac.authorization.k8s.io/awx-operator-awx-manager-rolebinding created
rolebinding.rbac.authorization.k8s.io/awx-operator-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/awx-operator-proxy-rolebinding created
configmap/awx-operator-awx-manager-config created
service/awx-operator-controller-manager-metrics-service created
deployment.apps/awx-operator-controller-manager created
```

### Modify kubectl Namespace and Wait for awx-operator Pod to be Running

The AWX kustomization file spins up AWX within a custom namespace called `awx`. Use the `kubectl config set-context --current --namespace=awx` command to modify the kubectl namespace to `awx`. Then, use the `kubectl get pods` command to confirm that the awx-operator pod is running and fully ready. This may take a few minutes to complete.

You can also use the `kubectl get pods -w` command to automatically watch for the awx-operator pod to be running and fully ready. Use Control+C to break out of this command once the awx-operator pod is running and fully ready.

```bash
kubectl config set-context --current --namespace=awx
kubectl get pods
```

Example output of these commands is below:

```console
christopher@awx:~$ kubectl config set-context --current --namespace=awx
Context "minikube" modified.

christopher@awx:~$ kubectl get pods
NAME                                               READY   STATUS              RESTARTS   AGE
awx-operator-controller-manager-566b76fc7f-4mjhr   0/2     ContainerCreating   0          7s

christopher@awx:~$ kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
awx-operator-controller-manager-566b76fc7f-4mjhr   2/2     Running   0          54s
```

### Create and Apply AWX Resource File

Create an AWX resource file, then modify the AWX kustomization file to include the AWX resource file as a valid resource. Then, apply the changes to the kustomization file with the `kubectl apply -k awx/` command.

```bash
cat <<EOF > awx/awx.yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
EOF
cat <<EOF > awx/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=2.5.3
  - awx.yaml

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.5.3

# Specify a custom namespace in which to install AWX
namespace: awx
EOF
kubectl apply -k awx/
```

Example output of these commands is below:

```console
christopher@awx:~$ cat <<EOF > awx/awx.yaml
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
EOF
christopher@awx:~$ cat <<EOF > awx/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=2.5.3
  - awx.yaml

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.5.3

# Specify a custom namespace in which to install AWX
namespace: awx
EOF
christopher@awx:~$ kubectl apply -k awx/
namespace/awx unchanged
customresourcedefinition.apiextensions.k8s.io/awxbackups.awx.ansible.com unchanged
customresourcedefinition.apiextensions.k8s.io/awxrestores.awx.ansible.com unchanged
customresourcedefinition.apiextensions.k8s.io/awxs.awx.ansible.com unchanged
serviceaccount/awx-operator-controller-manager unchanged
role.rbac.authorization.k8s.io/awx-operator-awx-manager-role configured
role.rbac.authorization.k8s.io/awx-operator-leader-election-role unchanged
clusterrole.rbac.authorization.k8s.io/awx-operator-metrics-reader unchanged
clusterrole.rbac.authorization.k8s.io/awx-operator-proxy-role unchanged
rolebinding.rbac.authorization.k8s.io/awx-operator-awx-manager-rolebinding unchanged
rolebinding.rbac.authorization.k8s.io/awx-operator-leader-election-rolebinding unchanged
clusterrolebinding.rbac.authorization.k8s.io/awx-operator-proxy-rolebinding unchanged
configmap/awx-operator-awx-manager-config unchanged
service/awx-operator-controller-manager-metrics-service unchanged
deployment.apps/awx-operator-controller-manager configured
awx.awx.ansible.com/awx created
```

### Watch Installation Process Through Logs

Use the `kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager` command to watch the logs of the AWX Operator spin up the AWX service. This may take several minutes to complete. Wait until the logs settle down and you see output similar to the following before proceeding:

```console
{"level":"info","ts":"2023-09-24T17:05:16Z","logger":"KubeAPIWarningLogger","msg":"unknown field \"status.conditions[1].ansibleResult\""}
```

```bash
kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager
```

```console
christopher@awx:~$ kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager
--------------------------- Ansible Task StdOut -------------------------------

 TASK [Remove ownerReferences reference] ********************************
ok: [localhost] => (item=None) => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}

-------------------------------------------------------------------------------
{"level":"info","ts":"2023-10-15T15:38:47Z","logger":"logging_event_handler","msg":"[playbook task start]","name":"awx","namespace":"awx","gvk":"awx.ansible.com/v1beta1, Kind=AWX","event_type":"playbook_on_task_start","job":"5219381047095076714","EventData.Name":"installer : Start installation if auto_upgrade is false and deployment is missing"}

--------------------------- Ansible Task StdOut -------------------------------

TASK [installer : Start installation if auto_upgrade is false and deployment is missing] ***
task path: /opt/ansible/roles/installer/tasks/main.yml:31

-------------------------------------------------------------------------------

----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----


PLAY RECAP *********************************************************************
localhost                  : ok=81   changed=0    unreachable=0    failed=0    skipped=81   rescued=0    ignored=1


----------
{"level":"info","ts":"2023-10-15T15:38:47Z","logger":"runner","msg":"Ansible-runner exited successfully","job":"5219381047095076714","name":"awx","namespace":"awx"}
{"level":"info","ts":"2023-10-15T15:38:47Z","logger":"KubeAPIWarningLogger","msg":"unknown field \"status.conditions[1].ansibleResult\""}
```

### Confirm Access to AWX Web Interface

This step will depend on whether you need to access the AWX web interface locally (meaning, from the same machine that is running Minikube) or remotely (meaning, from a different machine than the one running Minikube).

#### Access AWX Web Interface Locally

If you need to access the AWX web interface locally, execute the `minikube service awx-service -n awx` command to view the IP address and port you should use to access the AWX web interface.

```bash
minikube service awx-service -n awx
```

Example output of this command is below:

```console
christopher@awx:~$ minikube service awx-service -n awx
|------------|---------------|---------------|-----------------------------|
| NAMESPACE  | NAME          | TARGET PORT   | URL                         |
| ---------- | ------------- | ------------- | --------------------------- |
| awx        | awx-service   | http/80       | http://192.168.49.2:31961   |
| ---------- | ------------- | ------------- | --------------------------- |
🎉  Opening service awx/awx-service in default browser...
👉  http://192.168.49.2:31961
```

The above output indicates you would access the AWX web interface through IPv4 address `192.168.49.2` and port `31961`.

#### Access AWX Web Interface Remotely

If you need to access the AWX web interface remotely, you'll need to port forward the pod's web service port number to the external IP of the Minikube host. The below command exposes port `8080` on your Minikube cluster's external IP address. You can then access the AWX web interface by navigating to `http://<minikube-external-ip>:8080` in your browser.

```bash
kubectl port-forward svc/awx-service --address 0.0.0.0 8080:80 &> /dev/null &
```

Example output of this command is below:

```console
christopher@awx:~$ kubectl port-forward svc/awx-service --address 0.0.0.0 8080:80 &> /dev/null &
[1] 117088
christopher@awx:~$
```

### Get AWX User Interface Admin Password

Execute the `kubectl get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode; echo` command to retrieve the password for the AWX admin user (default username is `admin`) to log into the AWX web interface.

```bash
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode; echo
```

Example output of this command is below:

```console
christopher@awx:~$ kubectl get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode; echo
Tq6Erarc3LYlO8sUxzNBl0FdOBfDYUjK
christopher@awx:~$
```

The above output indicates the default password for the `admin` user account is `Tq6Erarc3LYlO8sUxzNBl0FdOBfDYUjK`. This password will change from one AWX installation to another.

AWX should now be installed and accessible through a local Minikube cluster on your Ubuntu 22.04 system.
