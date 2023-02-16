---
layout: post
title: netfilter hooks to snoop packets
date: '2014-03-24T07:47:00.001-07:00'
author: arges
tags:
- kernel
- networking
- module
modified_time: '2014-03-24T07:47:42.345-07:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-2554099113331291474
blogger_orig_url: http://dinosaursareforever.blogspot.com/2014/03/netfilter-hooks-to-snoop-packets.html
---

I've written a short netfilter hook to help snoop some outgoing packets. You can
view the source code [here][1]. I read [a][2] [few][3] [articles][4] to get an
idea of how to put things together.

[1]: https://github.com/arges/nf-snoop
[2]: http://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-3.html
[3]: http://www.linuxjournal.com/article/7184
[4]: http://www.paulkiddie.com/2009/10/creating-a-simple-hello-world-netfilter-module/
