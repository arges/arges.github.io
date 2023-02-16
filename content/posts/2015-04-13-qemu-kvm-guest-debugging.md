---
layout: post
title: qemu/kvm guest debugging
date: '2015-04-13T12:24:22-0500'
author: arges
tags:
- kvm
- ubuntu
- debug
modified_time: '2015-04-22T09:20:21-0500'

---

Occasionally it is useful to debug a running guest VM's kernel. Setup your host
machine for virtual machine hosting with QEMU/KVM, and add ddebs to your host
system using this [wiki][1].

Next, compile your own qemu with `--enable-debug`. Setup your environment to
use this binary (if using libvirt), or call it directly using qemu.

Ensure you have `vmlinux` of the L1 guest somewhere in your L0 machine.

First, setup your L1 guest. If running with qemu add the `-s` flag. With
libvirt you can add the following entry to your machine description:

~~~bash
# Modify this:
<domain type='kvm'>
# To look like this:
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <qemu:commandline>
    <qemu:arg value='-s'/>
  </qemu:commandline>
~~~

Start the guest and find the start address of the text section:

~~~bash
echo 0x$(sudo cat /proc/kallsyms | egrep -e "T _text$" | awk '{print $1}')
~~~

Next attach a gdb session on the running guest. Add the symbol file for vmlinux
using the proper start address. Then add a breakpoint where you want to detect
the failure. In this example we'll use `sysrq_handle_crash` which can be
triggered easily with `/proc/sysrq-trigger`.

~~~bash
gdb /usr/bin/qemu-system-x86_64
(gdb) target remote localhost:1234
(gdb) add-symbol-file vmlinux 0xffffffff81000000
(gdb) b sysrq_handle_crash
(gdb) c
~~~

In the VM run the following:

~~~bash
echo 'c' | sudo tee /proc/sysrq-trigger
~~~

Now you'll notice gdb has stopped at the breakpoint:

~~~bash
(gdb) c                                                              
Continuing.                                                          
[New Thread 2]                                                       
[Switching to Thread 2]                                              
                                                                     
Breakpoint 1, sysrq_handle_crash (key=99) at drivers/tty/sysrq.c:136 
136     drivers/tty/sysrq.c: No such file or directory.              
~~~

Additional information can be found [here][2] and [here][3].

[1]: https://wiki.ubuntu.com/DebuggingProgramCrash
[2]: https://blog.nelhage.com/2013/12/lightweight-linux-kernel-development-with-kvm/
[3]: http://www.linux-magazine.com/Online/Features/Qemu-and-the-Kernel

