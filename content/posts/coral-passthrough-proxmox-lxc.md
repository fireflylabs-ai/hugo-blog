---
title: "Coral Passthrough to Proxmox LXC"
date: 2022-10-17T20:13:03Z
draft: false
tags: [
    "coral",
    "lxc",
    "proxmox",
]
---

## Overview

In this article, I'll show you a quick and easy way to pass through a Google Coral USB Accelerator resource into an LXC container. This is useful if using detection in Frigate NVR or any other TensorFlow AI modeling.

## Prerequisites & Assumptions

This process assumes:

* You have an operational Proxmox & LXC setup running already. In this scenario, I'm using Ubuntu 22.04
* You understand common LXC terminology and can access the LXC command line interface in your container

## Environment Details

Proxmox Info:

``` shell
root@pve04:~# pveversion -v
proxmox-ve: 7.2-1 (running kernel: 5.15.60-2-pve)
pve-manager: 7.2-11 (running version: 7.2-11/b76d3178)
```

Additionally, a Google Coral USB plugged into a USB 3.0 port.

## Requirements

* You must pass the entire bus on which the Coral USB is running. You cannot only pass a single port when using Coral USB.
* Note that your bus id/port will change after the Coral USB initializes.

   For example, it will start as this in `lsusb`:

   `Bus 002 Device 002: ID 1a6e:089a Global Unichip Corp.`

   And end up as:

   `Bus 002 Device 003: ID 18d1:9302 Google Inc.`

## Process

The process to get this running is simple:

1. Create your CT on the host

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

2. Obtain USB bus info:

    ``` shell
    root@pve05:~# lsusb
    Bus 002 Device 002: ID 1a6e:089a Global Unichip Corp.
    Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
    ```

   In this case, you can see we are on USB Bus 002.

3. You will need to add the following LXC configuration options, matching `002` references in `lxc.mount.entry` with the bus ID you obtained in step 2:

    ``` shell
    lxc.apparmor.profile: unconfined
    lxc.cgroup2.devices.allow: a
    lxc.cap.drop: 
    lxc.mount.auto: cgroup:rw
    lxc.cgroup2.devices.allow: c 189:* rwm
    lxc.mount.entry: /dev/bus/usb/002 dev/bus/usb/002 none bind,optional,create=dir 0, 0
    ```

Now your Coral USB is available at `/dev/bus/usb/002` inside your CT!
