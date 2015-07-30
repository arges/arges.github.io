---
layout: post
title: using ktest.pl with ubuntu
date: '2014-07-01T13:10:00.002-07:00'
author: arges
tags:
- ktest
- kernel
- kvm
- ubuntu
modified_time: '2014-07-01T13:10:45.923-07:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-7189421098293937654
blogger_orig_url: http://dinosaursareforever.blogspot.com/2014/07/using-ktestpl-with-ubuntu.html
---

Bisecting the kernel is one of those tasks that's time-consuming and error prone.
Ktest.pl is a script that lives in the linux kernel source [tree][1] that helps
to automate this process. The script is extremely extensible and as
such takes time to understand which variables need to be set and where. In this
post, I'll go over how to perform a kernel bisection using a VM as the target
machine. In this example I'm using 'ubuntu' as the VM name.

First, ensure you have all dependencies correctly setup:

```bash
sudo apt-get install libvirt-bin qemu-kvm cpu-checker virtinst uvtool git
sudo apt-get build-dep linux-image-`uname -r`
```

Now, ensure kvm works: ```kvm-ok```

In this example we are using uvtool to create VMs using cloud images, but you
could just as easily use a preseed install or a manual install via an ISO.

Next sync the cloud image, and clone the necessary git directory:

```bash
uvt-simplestreams-libvirt sync release=trusty arch=amd64
git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git linux.git
```

Copy ktest.pl outside of the linux kernel (since bisecting it also changes the
script, this way it remains constant):

```bash
cd linux
cp tools/testing/ktest/ktest.pl ..
cp -r tools/testing/ktest/examples/include ..
cd ..
```

Create directories for script:

```bash
mkdir configs build
mkdir configs/ubuntu build/ubuntu
```

Get an appropriate config for the bisect you are using and ensure it can
reasonably `make oldconfig` with the kernel version you are using. For example,
if we are bisecting v3.4 kernels, we can use an Ubuntu v3.5 series kernel config
and ```yes '' | make oldconfig``` to ensure it is very close. Put this config
into ```configs/ubuntu/config-min```.

Create the VM, ensure you have ssh keys setup on your local machine first:

```bash
uvt-kvm create ubuntu release=trusty arch=amd64 --password ubuntu
```

Ensure the VM can be ssh'ed to via ```ssh ubuntu@ubuntu```:

```bash
echo "$(uvt-kvm ip ubuntu) ubuntu" | sudo tee -a /etc/hosts
```

SSH into the VM `with ssh ubuntu@ubuntu`.
Set up the initial target kernel to boot on the VM:

```bash
sudo cp /boot/vmlinuz-`uname -r` /boot/vmlinuz-test
sudo cp /boot/initrd.img-`uname -r` /boot/initrd.img-test
```

Ensure SUBMENUs are disabled on the VM, as the grub2 detection script in ktest.pl
fails with submenus, and update grub.

```bash
echo "GRUB_DISABLE_SUBMENU=y" | sudo tee -a /etc/default/grub
sudo update-grub
```

Ensure we have a serial console on the VM with `/etc/init/ttyS0.conf`, and ensure
that agetty automatically logs in as root. If you ran with the above script you
can do the following:

```bash
sudo sed -i 's/exec \/sbin\/getty/exec \/sbin\/getty -a root/' /etc/init/ttyS0.conf
```

Ensure that `/root/.ssh/authorized_keys` on the VM contains the host keys so that
```ssh root@ubuntu``` works automatically. If you are using the above commands
you can do:

```bash
sudo sed -i 's/^.*ssh-rsa/ssh-rsa/g' /root/.ssh/authorized_keys
```

Finally add a test case to `/home/ubuntu/test.sh` inside of the ubuntu VM. Ensure
it is executable.

```bash

#!/bin/bash
# Make a unique string
STRING=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c 32) > /var/log/syslog
echo $STRING > /dev/kmsg
# Wait for things to settle down...
sleep 5
grep $STRING /var/log/syslog
# This should return 0.
```

Now exit out of the machine and create the following configuration file for
ktest.pl called ubuntu.conf. This will bisect from v3.4 (good) to v3.5-rc1
(bad), and run the test case that we put into the VM.

```bash
# Setup default machine
MACHINE = ubuntu

# Use virsh to read the serial console of the guest
CONSOLE = virsh console ${MACHINE}
CLOSE_CONSOLE_SIGNAL = KILL

# Include defaults from upstream
INCLUDE include/defaults.conf
DEFAULTS OVERRIDE

# Make sure we load up our machine to speed up builds
BUILD_OPTIONS = -j12

# This is required for restarting VMs
POWER_CYCLE = virsh destroy ${MACHINE}; sleep 5; virsh start ${MACHINE}

# Use the defaults that update-grub spits out
GRUB_FILE = /boot/grub/grub.cfg
GRUB_MENU = Ubuntu, with Linux test
GRUB_REBOOT = grub-reboot
REBOOT_TYPE = grub2

DEFAULTS

# Do a simple bisect
TEST_START
RUN_TEST = ${SSH} /home/ubuntu/test.sh
TEST_TYPE = bisect
BISECT_GOOD = v3.4
BISECT_BAD = v3.5-rc1
CHECKOUT = origin/master
BISECT_TYPE = test
TEST = ${RUN_TEST}
BISECT_CHECK = 1

```

Now you are ready to run the bisection (this will take many, many hours depending on the
speed of your machine):

```bash
./ktest.pl ubuntu.conf
```

[1]: http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/tools/testing/ktest?id=HEAD

