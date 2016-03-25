---
layout: post
title: building ubuntu kernels with debug symbols
date: '2015-10-02T08:53:52-0500'
author: arges
tags:
- linux
- ubuntu
modified_time: '2015-10-02T08:53:56-0500'

---

Occasionally it is useful to be able to build a kernel the Ubuntu way with
debug symbols. The following is how to install dependencies, clone the tree,
and finally build in such a way that ddeb packages get generated.

Here's how:

~~~bash
sudo apt-get build-dep linux-image-$(uname -r)
sudo apt-get install fakeroot pkg-config-dbgsym git
git clone git://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux/+git/wily 
cd wily
fakeroot debian/rules clean
debian/rules build-generic
fakeroot debian/rules binary-generic binary-headers skipdbg=false
~~~

Wait a bit and you should see a -dbgsym package get generated.

