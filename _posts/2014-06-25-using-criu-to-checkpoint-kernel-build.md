---
layout: post
title: using CRIU to checkpoint a kernel build
date: '2014-06-25T13:46:00.001-07:00'
author: arges
tags:
- criu
- ubuntu
modified_time: '2014-06-25T13:46:19.387-07:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-5197537771410709807
blogger_orig_url: http://dinosaursareforever.blogspot.com/2014/06/using-criu-to-checkpoint-kernel-build.html
---

[CRIU][1] stands for Checkpoint/Restart in Userspace. As the criu package should
be landing in Utopic soon, and I wanted to test drive it to see how it handles.

I thought of an interesting example of being in the middle of a linux kernel
build and a security update needing to be installed and the machine rebooted.
While most of us could probably just reboot and rebuild, why not checkpoint it
and save the progress; then restore after the system update?  I admit its not
the most useful example; but it is pretty cool nonetheless.

~~~bash
sudo apt-get install criu
# start build; save the PID for later
cd linux; make clean; make defconfig
make & echo $! > ~/make.pid
# enter a clean directory that isn't tmp and dump state
mkdir ~/cr-build && cd $_
sudo criu dump --shell-job -t $(cat ~/make.pid)
# install and reboot machine
# restore your build
cd ~/cr-build
sudo criu restore --shell-job -t $(cat ~/make.pid)
~~~

And you're building again!

[1]: http://criu.org/Main_Page

