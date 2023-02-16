---
layout: post
title: creating a local vm easily and automatically
date: '2014-03-05T14:22:00.000-08:00'
author: arges
tags:
- kvm
- ubuntu
modified_time: '2014-03-05T14:39:00.307-08:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-3105487472221967322
blogger_orig_url: http://dinosaursareforever.blogspot.com/2014/03/creating-local-vm-easily-and.html
---

This is the nifty little snippet I use to create a quick and dirty VM for
testing. Just `apt-get install vmbuilder` and run the following:

~~~bash
NAME="ubuntu"
vmbuilder kvm ubuntu --arch 'amd64' --suite 'precise' \
--rootsize '8096' \
--mem '1024' \
--components 'main,universe' \
--addpkg vim  \
--addpkg openssh-server  \
--addpkg bash-completion  \
--user 'ubuntu'  --pass 'ubuntu' \
-d /var/lib/libvirt/images/${NAME} \
--libvirt qemu:///system \
-o \
-v --debug \
--hostname ${NAME}
~~~

If you need additional help consult the
[serverguide](https://help.ubuntu.com/12.04/serverguide/jeos-and-vmbuilder.html
)

