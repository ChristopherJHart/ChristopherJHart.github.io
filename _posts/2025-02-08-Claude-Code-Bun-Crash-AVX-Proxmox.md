---
layout: post
title: 'Claude Code Bun Crash - CPU Lacks AVX Support on Proxmox VM'
---

After a recent update, Claude Code's native installation method (which replaced the now-deprecated npm installation) began crashing on startup with the following error:

```
christopher@playground:~$ claude
============================================================
Bun Canary v1.3.9-canary.51 (d5628db2) Linux x64 (baseline)
Linux Kernel v6.8.0 | glibc v2.39
Features: jsc no_avx2 no_avx standalone_executable
Builtins: "bun:main"
Elapsed: 15017ms | User: 9445ms | Sys: 5472ms
RSS: 14.45GB | Peak: 5.24GB | Commit: 14.45GB | Faults: 212 | Machine: 16.77GB

CPU lacks AVX support. Please consider upgrading to a newer CPU.
panic(main thread): Segmentation fault at address 0x275E9800000
oh no: Bun has crashed. This indicates a bug in Bun, not your code.

Illegal instruction (core dumped)
```

Under the hood, Claude Code's native installation leverages [Bun](https://bun.sh/) as its JavaScript runtime. Bun requires CPUs that support [Advanced Vector Extensions (AVX)](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions), a set of x86 CPU instructions introduced by Intel in 2011. The crash output confirms this with the `CPU lacks AVX support` message and the `no_avx2 no_avx` feature flags.

## My Environment

In my case, Claude Code was running on an Ubuntu 24.04 virtual machine hosted on [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment/overview). The physical host's CPU *does* support AVX, but the VM was not configured to expose the host's CPU features to the guest.

I confirmed this by checking the CPU flags advertised to the VM:

```
christopher@playground:~$ cat /proc/cpuinfo | grep flags | head -1 | tr ' ' '\n' | grep avx
christopher@playground:~$
```

No output means no AVX support is visible to the guest operating system.

## Root Cause

Under my Proxmox VM's Hardware configuration, the Processors section had a Type of `Default (kvm64)`. The `kvm64` CPU type emulates a generic x86-64 CPU that does *not* include AVX extensions, regardless of whether the underlying physical host supports them.

## Fix

The fix is to change the Proxmox VM's CPU type to `host`, which passes the physical CPU's feature flags (including AVX) directly through to the VM.

1. Log into the Proxmox web interface.
2. Select the relevant virtual machine.
3. Navigate to the **Hardware** tab.
4. Double-click on the **Processors** row (or select it and click **Edit**).
5. Change the **Type** dropdown from `Default (kvm64)` to `host`.
6. Click **OK**.
7. **Shut down the VM from Proxmox** (not from within the guest) and start it again.

Step 7 is important. A guest-initiated reboot (e.g. `sudo shutdown -r now`) is *not* sufficient because the CPU type is a QEMU hardware definition that is applied when Proxmox launches the VM process. The guest operating system cannot pick up this change on a warm reboot.

After the VM started back up, I confirmed AVX support was now visible:

```
christopher@playground:~$ cat /proc/cpuinfo | grep flags | head -1 | tr ' ' '\n' | grep avx
avx
avx2
avx_vnni
christopher@playground:~$
```

Claude Code started without issue after this change.

## Applicability

This fix is specific to virtual machines hosted on Proxmox (or any QEMU/KVM-based hypervisor) where the CPU type is set to an emulated model that lacks AVX support. If you are running Claude Code on bare metal, within a container, or on a VM hosted on a different hypervisor, your root cause and fix may differ.

If your physical CPU genuinely does not support AVX (for example, CPUs manufactured before ~2011), changing the CPU type will not help. You would need hardware that supports the AVX instruction set.
