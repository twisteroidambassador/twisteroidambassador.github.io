---
layout: post
title: "NanoPi R4S: installing OpenWRT, LXC and others"
date: 2024-05-31 07:02:55 +0000
---

So, I got a NanoPi R4S.
I want to run OpenWRT and Unifi Controller (oh, it got renamed to Unifi Network Application) on it.
My plan is to install OpenWRT on it directly, and then run a LXC container with a full Linux distro to run Unifi.

This post will be a collection of problems encountered, and my solutions.

## Certain MicroSD cards won't boot: UHS?

The new MicroSD card I bought for the R4S does not boot OpenWRT 23.05.3.
The PWR light comes on, but all other LEDs remain off.

The problematic MicroSD card is a 64GB Sandisk Ultra MicroSDHC, with markings UHS-I U1 A1.
The same OpenWRT image when flashed on other, older MicroSD cards boots fine.

UART logs of the older cards booting:

```
U-Boot TPL 2021.07-OpenWrt-r23809-234f1a2efa (Mar 22 2024 - 22:09:42)
Channel 0: LPDDR4, 50MHz
BW=32 Col=10 Bk=8 CS0 Row=15 CS1 Row=15 CS=2 Die BW=16 Size=2048MB
Channel 1: LPDDR4, 50MHz
BW=32 Col=10 Bk=8 CS0 Row=15 CS1 Row=15 CS=2 Die BW=16 Size=2048MB
256B stride
lpddr4_set_rate: change freq to 400000000 mhz 0, 1
lpddr4_set_rate: change freq to 800000000 mhz 1, 0
Trying to boot from BOOTROM
Returning to boot ROM...

U-Boot SPL 2021.07-OpenWrt-r23809-234f1a2efa (Mar 22 2024 - 22:09:42 +0000)
Trying to boot from MMC1


U-Boot 2021.07-OpenWrt-r23809-234f1a2efa (Mar 22 2024 - 22:09:42 +0000)

SoC: Rockchip rk3399
Reset cause: POR
Model: FriendlyElec NanoPi R4S
DRAM:  3.9 GiB
PMIC:  RK808
MMC:   mmc@fe320000: 1
Loading Environment from MMC... MMC Device 0 not found
*** Warning - No MMC card found, using default environment

rk3399_vop vop@ff8f0000: failed to get ahb reset (ret=-524)
rk3399_vop vop@ff8f0000: failed to get ahb reset (ret=-524)
In:    serial
Out:   serial
Err:   serial
Model: FriendlyElec NanoPi R4S
Net:
Error: ethernet@fe300000 address not set.
No ethernet found.

Hit any key to stop autoboot:  0
MMC Device 0 not found
no mmc device at slot 0
switch to partitions #0, OK
mmc1 is current device
Scanning mmc 1:1...
Found U-Boot script /boot.scr
```

With the new card, the beginning part of boot log are identical, until:

```
Hit any key to stop autoboot:  0
MMC Device 0 not found
no mmc device at slot 0
starting USB...
Bus usb@fe380000: USB EHCI 1.00
```

[From the wiki article of Rockchip SOCs](https://opensource.rock-chips.com/wiki_Boot_option)
we know the boot rom is responsible for loading U-boot
(both TPL/SPL and U-boot proper, if I'm interpreting "Returning to boot ROM" correctly as "SPL_BACK_TO_BROM option enabled".)
Apparently, U-boot itself simply does not see the card at all.
All of this happens before the kernel is loaded, so it's not the same problem described
[here about the kernel not supporting UHS cards.](https://kohlschuetter.github.io/blog/posts/2022/10/28/linux-nanopi-r4s/)

The good news is, a snapshot image (as of 2024-05) boots up just fine with the new card.

But I want to use a release, not snapshot, on my device.
Thankfully the solution is simple:
Just transplant snapshot U-boot onto the release image.
[Again from the wiki article](https://opensource.rock-chips.com/wiki_Boot_option)
we know the SPL/TPL starts at byte 0x40 \* 512 = 0x8000, and the kernel starts at 0x8000 \* 512 = 0x1000000,
so just copy everything between those positions from the snapshot image to a release image.
Yes, it does work.

## Installing and running LXC

### Installation

Mostly, just follow OpenWRT's wiki article on [Running LXC on OpenWrt Host](https://openwrt.org/lxc_openwrt_host).
However, there is an important change:

**Do NOT install `cgroupfs-mount`**.

Without it, the stock OpenWRT 23.05 system already mounts a cgroup2 file system, and it's usable by LXC.
`cgroupfs-mount` will unmount the cgroup2 fs and mount a bunch of cgroup (v1) file systems, which then stop LXC from working.

### cgroup controllers

When starting an LXC container in the foreground, the first lines printed to the console are the same error message repeated twice:

```
lxc-start: <container name>: ../src/lxc/cgroups/cgfsng.c: __cgfsng_delegate_controllers: 3341 Invalid argument - Could not enable "+cpuset +cpu +io +memory +pids +rdma" controllers in the unified cgroup 7
```

And when we do `ls /sys/fs/cgroup/lxc.payload.<container name>/`, we realize all the controllers are missing, and we can't set any resource limits.

We try adding the controllers to the root cgroup, and find only one causing trouble:

```
root@OpenWrt:~# echo "+cpuset" > /sys/fs/cgroup/cgroup.subtree_control 
ash: write error: Invalid argument
```

Unfortunately,
[LXC only adds all the controllers in one go](https://github.com/lxc/lxc/blob/c44f424f9b0fb84945a554a6ccbdf5cde0780cf5/src/lxc/cgroups/cgfsng.c#L3574-L3592),
so one controller failing to add means we have no controller at all.

The problem might be this:
[EINVAL when enabling cpuset in cgroupv2 when io_uring
worker threads are running](https://lore.kernel.org/io-uring/CA+wXwBQwgxB3_UphSny-yAP5b26meeOu1W4TwYVcD_+5gOhvPw@mail.gmail.com/).
I could not find a way to verify the cause, but it does give a clue:
If something running is in conflict with enabling cpuset, maybe we can enable it before whatever conflicting thing runs?

And indeed, it suffices to enable the controller in `rc.local`.

Just add the following snippet to `/etc/rc.local` to enable all available controllers at boot time:

```
for controller in $(cat /sys/fs/cgroup/cgroup.controllers); do
    echo "+${controller}" > /sys/fs/cgroup/cgroup.subtree_control
done
```

After a quick reboot, `/sys/fs/cgroup/cgroup.subtree_control` contains all available controllers, and there are no more error messages starting a container.