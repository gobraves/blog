---
layout: post
title: Compile the Linux Kernel
date: 2016-03-23
tags: Note
categories: Linux
---

### Compile the Linux Kernel 

1. download the linux kernel source code
2. cd the source code dir
3. `make distclean`
4. `make menuconfig` or `make oldconfig`
5. `make all -j4` or `make` then `make modules_install`
6. `cp -v arch/x86_64/boot/bzInage /boot/vmlinuz-linux-version`
7. `cp system.map /boot/System.map-version`
8. `cp .config /boot/config-version`
9. `update-grub`
10. `reboot`

> `bindeb-pkg` or `binrpm-pkg` could substitute step 5 to step 9. Then `dpkg -i kernel.deb` or `rpm -i kernel.rpm`


