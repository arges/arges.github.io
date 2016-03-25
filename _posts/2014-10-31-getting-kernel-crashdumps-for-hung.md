---
layout: post
title: getting kernel crashdumps for hung machines
date: '2014-10-31T13:53:00.001-07:00'
author: arges
tags:
- linux
- howto
- debugging
- ubuntu
modified_time: '2014-10-31T13:53:55.494-07:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-2109760927377215205
blogger_orig_url: http://dinosaursareforever.blogspot.com/2014/10/getting-kernel-crashdumps-for-hung.html
---

Debugging hung machines can be a bit tricky. Here I'll document methods to
trigger a crashdump when these hangs occur.  What exactly does it mean when a
machine 'hangs' or 'freezes-up'? More information can be found in the kernel
[documentation][1], but overall there are really a few types of hangs.

A "Soft Lock-Up" is when the kernel loops in kernel mode for a duration without
giving tasks a chance to run. A "Hard Lock-Up" is when the kernel loops in kernel
mode for a duration without letting other interrupts run. In addition a
"Hung Task" is when a userspace task has been blocking for a duration.

Thankfully the kernel has options to panic on these conditions and thus create a
proper crashdump. In order to setup crashdump, on an Ubuntu machine we can do
the following. First we need to install and setup crashdump, more info can be
found [here][2]

~~~bash
sudo apt-get install linux-crashdump
~~~

Select NO unless you really would like to use kexec for your reboots.
Next we need to enable it since by default it is disabled.

~~~bash
sudo sed -i 's/USE_KDUMP=0/USE_KDUMP=1/' /etc/default/kdump-tools
~~~

Reboot to ensure the kernel cmdline options are properly setup

~~~bash
sudo reboot
~~~

After reboot run the following:

~~~bash
sudo kdump-config show
~~~

If this command shows 'ready to dump', then we can test a crash to ensure kdump
has enough memory and will dump properly.  This command will crash your
computer, so hopefully you are doing this on a test machine.

~~~bash
echo c | sudo tee /proc/sysrq-trigger
~~~

The machine will reboot and you'll see a crash in /var/crash.  All of this is
already documented [here][2], so now we need to enable panics for hang and
lockup conditions.  Now we need to enable crashing on lockups, so we'll enable
many cases at once.  Edit /etc/default/grub and change this line to the
following:

~~~bash
GRUB_CMDLINE_LINUX="nmi_watchdog=panic hung_task_panic=1 softlockup_panic=1 unknown_nmi_panic"
~~~

In addition you could enable these via /proc/sys/kernel or sysctl. For more
information about these parameters there is documentation [here][3] If you've
used the command line change, update grub and then reboot.

~~~bash
sudo update-grub
sudo reboot
~~~

Now your machine should crash when it locks up, and you'll get a nice crashdump
to analyze. If you want to test such a setup I wrote a [module][4] that induces
a hang to see if this works properly.

[1]: https://www.kernel.org/doc/Documentation/lockup-watchdogs.txt
[2]: https://wiki.ubuntu.com/Kernel/CrashdumpRecipe
[3]: https://www.kernel.org/doc/Documentation/kernel-parameters.txt
[4]: https://github.com/arges/hanger

