---
layout: post
title: "Making the ZTE MF910 play nice"
description: ""
category: 
tags: [mf910, Linux]
---
{% include JB/setup %}

The ZTE MF823/831/910 modems/mifis are of the type that will just work with
Linux, or at least should. They are all exposed as either RNDIS or CDC Ethernet
devices and appear on the host as a "normal" network interface. In addition,
most of the configuration is done automatically. In other words, most users will
just plug them in and all is good. And if you need to change the configuration,
then a pretty ok web user interface/REST API is provided.

In the [MONROE EU project](https://www.monroe-project.eu/), which is going to
develop a platform for measuring mobile broadband in Europe, they have selected
the MF910 as their main modem for use by experimenters. Each measurement node
will be multihomed and equipped with several modems, WAN port, etc. My company,
[Celerway](http://celerway.com/), is one of the partners in the project and we
were tasked with taking a closer look at the modem and make it work properly.
Turned out we had our work cut out for us, the MF910 has some ... interesting
stability issues and quirks.

### Improving stability

The MF910 is mostly a very stable device. We pushed the mifi very hard, used it
in rough conditions, etc., and the only times the device has crashed has been
because of failures in external components. For example, we had some what turned
out to be bad converters (for use with mobile measurement nodes), and they were
not able to stay within the +/- 5% USB 2.0 limit on input voltage. When this
occurred, dmesg was filled with different USB errors (typically -32) and we had
to power-cycle the device for it to appear again.

However, we frequently saw the device crash when it was switched from storage
mode and into modem mode. The MF910, as several other modems, are initially
exposed as a storage device and then switched into modem mode. The latter is
typically done by usb\_modeswitch, but the MF910 is handled by a legacy
in-kernel modeswitch in usb-storage (see "Kernel related issues"
[here](http://www.draisberghof.de/usb_modeswitch/) for more info). Since we do
not use USB storage, we disabled the module and used usb\_modeswitch for
switching the mifi into modem mode. After this change, we have not lost any
devices when switching. If you need USB storage, the page that I link to above
has some suggestions.

### Static MAC address part 1

The MF910 mifi and the 823/831 modems use a secret OS-fingerprinting algorithm
for deciding how to expose the device. We have seen the devices as both RNDIS
and CDC Ethernet, as mentioned earlier. The most common mode is RNDIS and, with
one exception, everything works as it should. Every device is exported with the
same MAC address, which can lead to all sorts of issues. Fortunately this
behavior was easy to sort out and our patch for the rndis\_host driver is
already [accepted into the Linux
kernel](http://git.kernel.org/cgit/linux/kernel/git/davem/net-next.git/commit/?id=a5a18bdf7453d505783e40e47ebb84bfdd35f93b).

### Static MAC address part 2

After the positive experience with RNDIS, we expected CDC Ethernet to be smooth
sailing. That would of course have been too easy and I think we got to see the
worst side of the MF910.

As with RNDIS, the device is exported with a static MAC so we started out by
setting a valid random MAC address on the interface. This seemingly worked fine
and we could communicate with the MF910 through the REST API. However, we were
unable to receive any traffic from external networks.

Some head-scratching and tcpdumping later, and we found the reason. For some
reason only known to the authors of the firmware, all packets routed through the
device (to the host) are sent with the same static destination MAC. We solved
this by adding an RX fixup function to the cdc\_ether-driver. The patch has
already been through several revisions and is expected to be accepted into the
kernel very soon. Until then, the latest version can be found
[here](https://patchwork.ozlabs.org/patch/651078/).

Unfortunately, this firmware bug makes it impossible to use for example macvlan
with the MF910 when it is exposed as CDC Ethernet. A different way to desciber
macvlan is MAC multiplexing, which does not work when the device only sends
packets with one destination MAC.

### Controlling which mode is exposed

In Monroe, each experiment is run in a container and we use macvlan to provide
internet connectivity. It is seemingly random which type of the device the MF910
is exposed as, which has led to some very weird bugs/behavior. Fortunately, it
turned out to be easy to switch the MF910 into the desired type of device (RNDIS
in our case). The following hidden URL does the job:

`/goform/goform_set_cmd_process?goformId=USB_MODE_SWITCH&usb_mode=X`

Change X to be the value matching the desired mode. 1-4 is RNDIS (we have not
found any difference between them), 5 is CDC Ethernet and 6 is ADB. Yes, you can
access the device using adb and hack around as much as you want. A factory reset
seems to reset any major breakage.
