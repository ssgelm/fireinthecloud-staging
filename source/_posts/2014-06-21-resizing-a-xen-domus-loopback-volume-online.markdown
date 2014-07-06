---
layout: post
title: "Resizing a Xen DomU's Loopback Volume Online"
date: 2014-06-21 16:17:19 -0500
comments: true
categories: xen
description: "How to resize a loopback volume that is attached to a Xen DomU without rebooting the DomU"
keywords: "xen, resize, online, loopback, online resize, domu resize, resize volume"
---

At Braintree we use Xen running on Debian for virtualization.  We have two generations of Dom0s.  Our newer generation uses LVM for VM storage and our older generation uses disk images attached to the VMs via loopback devices.  We often have the need to enlarge a disk volume attached to a DomU.  In order to maintain the highest uptime possible we prefer to resize the VM online without needing to shutdown or even reboot the VM.  All of our services are highly available so they can tolerate a reboot of a single VM but we still try not to rely on that if possible.  Resizing an LVM-backed volume online is a pretty simple process as long as you are presenting the volume to the DomU as a partition instead of an entire disk (just `lvresize` it on the Dom0 then `resize2fs` it on the DomU).  Pretty much [all](http://theninthdimension.blogspot.com/2010/01/how-to-resize-grow-xen-vm-partition.html) [articles](http://grantmcwilliams.com/item/262-resize-disk-image-used-as-xen-domu-hard-drive) [about](http://amandine.aupetit.info/187/add-disk-space-to-a-img-disk-image-for-use-with-xen-for-example/) resizing a Xen loopback volume list the first step as shutting down the DomU.  [Some](http://serverfault.com/questions/65340/how-do-i-get-a-xen-domu-to-notice-a-change-in-the-block-size-of-one-of-its-phy-d) go as far as to claim it's impossible to resize it online.

<!-- more -->

Undeterred, I experimented with some methods to resize the disk online and figured it out!  There are a few caveats to this method:

  * This will only work with raw disk images.  It may be possible to resize QCOW2 and VHD disk images using a similar process (with the appropriate tool instead of `dd`) but I have not tested it.
  * This assumes that the disk image is exposed to the vm as a partition (e.g. /dev/xvda1) and that the image itself is not partitioned.  Getting linux to re-read the partition table of a disk that has mounted partitions without a reboot is unreliable at best but is sometimes possible.
  * This should go without saying but _always_ back up your data before doing this.  I have sucessfully used it a number of times however resizing a disk is always inherently risky and doing so online is even more so.  Better safe than sorry.
  * I have confirmed that this works on Xen 4.0 running on Debian Squeeze.  Newer versions of Xen should work fine but older versions may not (`xm block-configure` was broken in Xen for a long time which prevents this process from working).

Resizing the disk
-----------------

In short, the process involves expanding the disk image, resizing the loopback device, telling xend to reconfigure the block device, then resizing the filesystem on the DomU.

* On the Dom0
    1. Use `dd` to expand the disk image, replacing `IMAGEFILENAME` with the path to the image you are resizing and `nn` with the number of GB you want to add to the volume (i.e. 50K for 50 GB):

        ```sudo dd if=/dev/zero of=IMAGEFILENAME bs=1M count=nnK oflag=append conv=notrunc```

    2. Next we need to resize the loopback device.

        1. Get the name of the loopback device Xen is using, again replacing `IMAGEFILENAME` with the path to the image you want to resize:

            ```sudo losetup -j IMAGEFILENAME```

        2. Resize the loopback device you just found.  Replace `LOOPDEV` with the loopback device the previous command returned (should be something like `/dev/loop1`):

            ```sudo losetup -c LOOPDEV```

    3. Now we need to tell Xen to reread the loopback device size.  Replace DOMUNAME with the name of the Xen DomU, IMAGEFILENAME with the full path to the disk image you have resized, and XENBLOCKDEV with the block device on the DomU (something like `xvda1`).  (Note that in every case I have tried it this command has returned an error saying `Error:` `Refusing to reconfigure device` even when it works successfully.  Go figure.):

        ```sudo xm block-configure DOMUNAME file:IMAGEFILENAME XENBLOCKDEV w```

* On the DomU
    1. First, make sure `parted` is installed.  In the case of Debian-based distros you can just run:

        ```sudo apt-get install parted```

    2. Now the OS needs to be told to reread the partition table:

        ```sudo partparobe```

    3. At this point you should be able to see that the partition is larger if you run `sudo fdisk -l /dev/XENBLOCKDEV` (replacing the appropriate block device).  You can now resize it (replacing `XENBLOCKDEV` with the appropriate device):

        ```sudo resize2fs /dev/XENBLOCKDEV```

If you do a `df` now you should see that the partition has, in fact, been resized.
