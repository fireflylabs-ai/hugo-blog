---
title: "GPU Passthrough to Proxmox LXC"
date: 2022-10-17T18:46:25Z
draft: false
tags: [
    "coral",
    "lxc",
    "proxmox",
]
---

## Overview

In this article, I'll show you a quick and easy way to pass through Intel iGPU resources into an LXC container. This is useful if using Plex, Frigate, Jellyfin, etc inside LXC on Proxmox.

## Prerequisites & Assumptions

This process assumes:

* You have an operational Proxmox & LXC setup running already. In this scenario, I'm using Ubuntu 22.04
* You understand common LXC terminology and can access the LXC command line interface in your container

## Environment Details

CPUs tested:

* Intel(R) Core(TM) i7-7700 CPU @ 3.60GHz (1 Socket) - `HD Graphics 630`
* Intel(R) Core(TM) i7-4790 CPU @ 3.60GHz (1 Socket) - `Xeon E3-1200 v3/4th Gen Core Processor Integrated Graphics Controller`

Proxmox Info:

``` shell
root@pve04:~# pveversion -v
proxmox-ve: 7.2-1 (running kernel: 5.15.60-2-pve)
pve-manager: 7.2-11 (running version: 7.2-11/b76d3178)
```

## Process

I'll break this information down into two pieces - things you need to do on the host, and an optional step you can do inside the LXC "guest".

### Host

Update `apt`'s cache, and install `va-drivers-all` and `vainfo`:

``` bash
apt update && apt install -y va-drivers-all vainfo
```

Update `grub`:

``` bash
update-grub
update-initramfs -u
```

Finally, reboot for changes to take effect:

``` bash
reboot
```

After reboot, `vainfo` should show proper GPU details:

``` shell
root@pve04:~# vainfo
error: XDG_RUNTIME_DIR not set in the environment.
error: can't connect to X server!
libva info: VA-API version 1.10.0
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_10
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.10 (libva 2.10.0)
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 21.1.1 ()
vainfo: Supported profile and entrypoints
      VAProfileMPEG2Simple            : VAEntrypointVLD
      VAProfileMPEG2Main              : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointEncSliceLP
      VAProfileH264High               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointEncSliceLP
      VAProfileJPEGBaseline           : VAEntrypointVLD
      VAProfileJPEGBaseline           : VAEntrypointEncPicture
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSliceLP
      VAProfileVP8Version0_3          : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileHEVCMain10             : VAEntrypointVLD
      VAProfileVP9Profile0            : VAEntrypointVLD
      VAProfileVP9Profile2            : VAEntrypointVLD
```

> NOTE: Your output may differ based on your iGPU's capabilities. This output is from the i7 7700, which supports `iHD` driver (instead of the less-capabale `i965` or `i915` driver). This is entirely controlled by your iGPU's capabilities and generation.

### CT Creation on Host

Create a CT via Ubuntu 22.04 Template, and do not yet boot it. Config should look like this:

``` shell
root@pve04:~# cat /etc/pve/lxc/3004.conf
arch: amd64
cores: 8
features: nesting=1
hostname: myubuntulxc
memory: 16384
nameserver: 10.1.10.2 10.1.10.3
net0: name=eth0,bridge=vmbr10,gw=10.1.10.1,hwaddr=D2:44:DD:EE:7F:B5,ip=10.1.10.64/24,type=veth
ostype: ubuntu
rootfs: lvm-thin-local:vm-3004-disk-0,size=16G
searchdomain: lan
swap: 0
```

> NOTE: Some details may be different for your network configuration, storage config, etc

You will need to add the following LXC configuration options:

``` shell
lxc.apparmor.profile: unconfined
lxc.cgroup.devices.allow: a
lxc.cap.drop: 
lxc.mount.auto: "proc:rw sys:rw"
lxc.cgroup2.devices.allow: c 226:0 rwm
lxc.cgroup2.devices.allow: c 226:128 rwm
lxc.cgroup2.devices.allow: c 29:0 rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
lxc.mount.entry: /dev/fb0 dev/fb0 none bind,optional,create=file
```

### Inside CT "guest"

Boot your new LXC container, and grab a terminal nside the lxc container and run the following:

Update `apt`'s cache, and install `vainfo`:

``` bash
apt update && apt install -y vainfo
```

Now you can view the output of `vainfo` as you did on the host to confirm the guest has full control over the Intel GPU:

``` shell
root@myubuntulxc:~# vainfo
error: can't connect to X server!
libva info: VA-API version 1.14.0
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_14
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.14 (libva 2.12.0)
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 22.3.1 ()
vainfo: Supported profile and entrypoints
      VAProfileMPEG2Simple            : VAEntrypointVLD
      VAProfileMPEG2Main              : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointEncSliceLP
      VAProfileH264High               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointEncSliceLP
      VAProfileJPEGBaseline           : VAEntrypointVLD
      VAProfileJPEGBaseline           : VAEntrypointEncPicture
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSliceLP
      VAProfileVP8Version0_3          : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileHEVCMain10             : VAEntrypointVLD
      VAProfileVP9Profile0            : VAEntrypointVLD
      VAProfileVP9Profile2            : VAEntrypointVLD
```

> NOTE: you can also run tools like `intel_gpu_top` (from the `intel-gpu-tools` package) to view real-time GPU usage, similar to what Nvidia provides with `nvidia-smi`.
