---
layout: post
title: "Help! Grub is installed on the wrong partition"
description: "Moving grub to the right partition"
category: Linux
tags: [Linux]
---
{% include JB/setup %}

Lately, I have frequently experienced that Grub is installed to the "wrong"
partition. Or, more precisely, that another device (typically the USB drive I
use to install different Linux distributions from) has given the highest
priority (lowest index) by the BIOS. This causes for example the
Ubuntu-installation to install Grub to the incorrect partition.

Fortunately, installing Grub on the correct hard drive is a straight-forward
procedure. This description is for Ubuntu, but a rescue/recovery mode exists for
the other distributions I have tried:

* Create a new startup disk, if the bootloader on your USB stick has been
  overwritten by Grub during installation.
* On boot, enter rescue/recovery (or similar) mode and drop to a shell. If you have option of
  using your main HDD as root file system, select that one.
* Run fdisk /dev/sdX (where X is the letter for your HDD). If you are unsure, it
  can be found by looking at the output of `fdisk -l`.
* Set the boot flag on the first partition and save the new partition table. The
  correct order of commands is `a` (for setting the bootable flag), `1` (select
  partition) and then `w` for writing.
* Then Grub needs to be installed. Run `grub-install /dev/sdX`.
* Reboot and all should be well.
