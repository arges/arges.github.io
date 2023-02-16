---
layout: post
title: creating a local vm domain from an ubuntu cloud image
date: '2014-02-17T11:43:00.003-08:00'
author: arges
tags:
- cloud
- kvm
- ubuntu
modified_time: '2014-03-05T14:37:23.635-08:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-887796959855210002
blogger_orig_url: http://dinosaursareforever.blogspot.com/2014/02/creating-local-vm-domain-from-ubuntu.html
---

It is very convenient to be able to quickly spin up a local VM in order to do
some sort of bug verification or testing. Generally I can use a cloud provider
to spin up an instance, but occasionally you need the flexibility of a local VM
in order to setup more complex networking or other specific environments.

This [link][1] was extremely helpful in figuring our how to setup a ready to boot
image, but I noticed a few issues. First, on Precise I didn't have the
cloud-localds tool, and second there wasn't a very simple way to import the
image into libvirt.

The cloud-localds problem was solved easily by branching the latest copy and
using it locally. Virt-install can be easily used to create a domain based on
the image.

Below is a snippet to be run after using Scott's original script:

~~~bash
# install cloud image
sudo virt-install --connect qemu:///system \
  --ram 1024 -n ubuntu --os-type=linux --os-variant=ubuntuprecise \
  --disk path=./disk.img,device=disk,bus=virtio,format=qcow2 \
  --disk path=./my-seed.img,bus=virtio,format=raw \
  --vcpus=1 --vnc --noautoconsole --import
~~~

[1]: http://ubuntu-smoser.blogspot.com/2013/02/using-ubuntu-cloud-images-without-cloud.html

