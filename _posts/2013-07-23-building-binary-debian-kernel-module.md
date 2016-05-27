---
layout: post
title: building a binary debian kernel module package with dkms
date: '2013-07-23T14:40:00.000-07:00'
author: arges
tags:
- debian
- dkms
- ubuntu
modified_time: '2013-09-10T09:32:40.782-07:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-6484236259965645454
blogger_orig_url: http://dinosaursareforever.blogspot.com/2013/07/building-binary-debian-kernel-module.html
---

DKMS packaging works great for building out of tree kernel modules. However,
what do you do when you need to install to a machine without a compiler? You
can accompish this by using DKMS's `mkdriverdisk` functionality.

First follow the steps [here][1] for setting up a proper DKMS package.

After you've built the module successfully, you can use the following bash
script to extract the .deb installer from the driverdisk. This way you can copy
the deb file to the target machine and install. You must ensure that `$(uname -rm)`
on the target machine matches the build machine.

~~~bash
NAME=hello
VERSION=0.1

# Build in a temp dir
TMPDIR=$(mktemp -d)
cd $TMPDIR
sudo dkms mkdriverdisk $NAME/$VERSION -d ubuntu --media tar | tee build.log
TARBALL_PATH=$(grep "Disk image location" build.log | cut -d':' -f2)
echo $TARBALL_PATH
tar -xf $TARBALL_PATH
cd -

# Extract the build
mv $TMPDIR/ubuntu-drivers/*/*.deb .
~~~

[1]: https://wiki.ubuntu.com/Kernel/Dev/DKMSPackaging

