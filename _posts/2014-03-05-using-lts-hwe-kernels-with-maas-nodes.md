---
layout: post
title: using LTS HWE kernels with MAAS nodes
date: '2014-03-05T14:21:00.002-08:00'
author: arges
tags:
- maas
- ubuntu
modified_time: '2014-03-05T14:38:15.723-08:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-6716830670481592950
blogger_orig_url: http://dinosaursareforever.blogspot.com/2014/03/using-lts-hwe-kernels-with-maas-nodes.html
---

Once you get machines setup using [MAAS][1], you may want to be able to install
an alternative kernel when starting a machine. Here's how to do it.

This is assuming we have already commissioned a node, but have not started it.

If we are using the _normal_ installer with MAAS do the following: Edit
`/etc/maas/preseeds/preseed_master` on your maas-server

Add the second line as shown:

~~~bash
d-i base-installer/kernel/image string linux-server d-i
base-installer/kernel/override-image string linux-generic-lts-saucy
~~~

Start the node, Now when you boot you should be using the 3.11 series kernel.

[1]: http://maas.ubuntu.com/
[2]: http://maas.ubuntu.com/docs/configure.html#altering-the-preseed-file


