---
type: post
title: using linux livepatch on ubuntu
date: '2015-09-21T13:03:26-0500'
author: arges
tags:
- linux
- ubuntu
- livepatch
- kpatch
modified_time: '2015-09-21T13:03:31-0500'

---

Livepatching was [introduced][2] in the v4.0 kernel, and now Ubuntu 15.10 has
a kernel capable of using this new and exciting feature. This works by using
ftrace to redirect kernel function calls to the newly patched functions. In
addition mechanisms for hooking into module insertion and removal are used for
patching loadable modules. This feature also has sysfs directories for tracking
which patches are applied and which functions they modify. With the basics
aside, this blog post will show some simple examples of how to livepatch your
kernel.

### Setup

If you are running the latest Ubuntu release livepatching will work as the
default kernel config has this enabled.

Next you'll need to ensure you have headers and debug symbols. This allows us
to build the kernel and download the original vmlinux for kpatch.

Note, if you are running an older kernel the debug symbols may have been
moved from the archive. Therefore you'll need to grab the debug symbols from
Launchpad. Start [here][4] and navigate to your release version and locate
the proper linux-image ddeb package.

If the ddebs are still available in the archive, you can do the following. Keep
in mind downloading the ddeb package takes a bit of time.

~~~bash
echo "deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse" | sudo tee -a /etc/apt/sources.list.d/ddebs.list
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 428D7C01
sudo apt-get update
sudo apt-get install linux-image-`uname -r`-dbgsym
~~~

Also be sure to install dependencies for building the kernel, kpatch, and the
kernel headers (in case they aren't already installed).

~~~
sudo apt-get install linux-headers-`uname -r`
sudo apt-get build-dep linux
sudo apt-get install git make gcc libelf-dev dpkg-dev
~~~

### Simple Example

Before using kpatch I'll show a simple example of creating a basic module. In
the kernel sources, there is a sample file which can be built [here][5]. Using
that patch we can create a simple example.

~~~
wget http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/plain/samples/livepatch/livepatch-sample.c
~~~

Create a Makefile as follows:

~~~bash
obj-m := livepatch-sample.o
KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(shell pwd)
default:
        $(MAKE) -C $(KDIR) SUBDIRS=$(PWD) modules
clean:
        $(MAKE) -C $(KDIR) SUBDIRS=$(PWD) clean
~~~

Make the module, test before, insert the module and test after
insertion:

~~~bash
$ make
make -C /lib/modules/4.2.0-10-generic/build SUBDIRS=/home/ubuntu/experiment modules
make[1]: Entering directory '/usr/src/linux-headers-4.2.0-10-generic'
  CC [M]  /home/ubuntu/experiment/livepatch-sample.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /home/ubuntu/experiment/livepatch-sample.mod.o
  LD [M]  /home/ubuntu/experiment/livepatch-sample.ko
make[1]: Leaving directory '/usr/src/linux-headers-4.2.0-10-generic'
$ l
livepatch-sample.c  livepatch-sample.ko  livepatch-sample.mod.c  livepatch-sample.mod.o  livepatch-sample.o  Makefile  modules.order  Module.symvers
$ cat /proc/cmdline 
BOOT_IMAGE=/boot/vmlinuz-4.2.0-10-generic root=UUID=2310261a-1c3b-476e-80ab-b14f12fd334f ro
$ sudo insmod livepatch-sample.ko 
$ cat /proc/cmdline 
this has been live patched
$ lsmod | grep livepatch
livepatch_sample       16384  1
~~~

Note, this module is in use once is it inserted. This is a hardcoded value
because the current version does not have a [consistency][6] model which will
enable removal of kernel patches. The reason for this restriction is safety
when removing a kernel patch.

### Kpatch Example

Next we'll use [kpatch][1] to generate a livepatch module. This can be very
useful if you're having to patch, for example, an inline function. Using the
above method you'd need to account for every parent function where that inlining
occurs and then copy the text for that code in its entirety. Kpatch, or more
specifically kpatch-build, takes a different approach building a non-patched
and patched kernel and looking for differences at the object level. Then kpatch
turns that into an actual kernel module compatible with livepatch.

Let's checkout and build the project:

~~~bash
git clone https://github.com/dynup/kpatch.git
cd ../kpatch && make
~~~

In this example, we'll patch the Ubuntu kernel sources. Therefore the first step
would be to clone a git tree with our current kernel. Next we make a
modification and export the diff.

~~~bash
git clone git://kernel.ubuntu.com/ubuntu/ubuntu-wily.git
cd ubuntu-wily
# make modifications to a file
git diff > ~/mypatch.diff
~~~

Now we can feed this into kpatch-build to create a compatible livepatch module.
Using the below command will automatically do the right thing for the installed
kernel on Ubuntu.

I added the `--skip-gcc-check` argument to kpatch-build, but normally you'd
want to ensure you were using the same version of the toolchain. You should
compare to see if there was a major version change and keep in mind this could
cause unwanted failures.

~~~bash
$ ./kpatch-build/kpatch-build -t vmlinux --skip-gcc-check meminfo-string.patch

WARNING: Skipping gcc version matching check (not recommended)
Using cache at /home/ubuntu/.kpatch/src
Testing patch file
checking file fs/proc/meminfo.c
Building original kernel
Building patched kernel
Detecting changed objects
Rebuilding changed objects
Extracting new and modified ELF sections
meminfo.o: changed function: meminfo_proc_show
Building patch module: kpatch-mypatch.ko
SUCCESS

$ sudo insmod kpatch-mypatch.ko
~~~

It works!

[1]: https://github.com/dynup/kpatch
[2]: https://lkml.org/lkml/2014/11/6/387
[3]: https://github.com/dynup/kpatch#ubuntu-1404
[4]: https://launchpad.net/ubuntu/wily/+source/linux
[5]: https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/samples/livepatch
[6]: https://lwn.net/Articles/632582/
