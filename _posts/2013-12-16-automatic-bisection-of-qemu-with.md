---
layout: post
title: automatic bisection of qemu with console and monitor interaction
date: '2013-12-16T14:39:00.001-08:00'
author: arges
tags:
- kvm
- bisect
- ubuntu
modified_time: '2014-03-05T14:36:53.073-08:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-176522600624306544
blogger_orig_url: http://dinosaursareforever.blogspot.com/2013/12/automatic-bisection-of-qemu-with.html
---

I had some fun debugging a performance [issue][1] after VM migration on v1.0.
Because this was fixed in newer versions, a bisect was in order. Doing this
manually becomes very tedious so writing some scripts to use ```git bisect run```
really helps. Here I'll document how I did this in case others find it useful
when trying to bisect issues in qemu.

First step is getting the problem reproducible using the command line. In
addition you will also need to output anything that be used to determine if the
test case passes or fails using standard console output. This makes it easy to
run the test case with expect. You will need to [follow][2] on how to setup your
VM to use serial output.

Next modify the expect script and ensure it works by just running it by itself.
Identify which versions pass or fail (a coarse bisect). Once you can get
between two release tags you should start a bisect between those tags.

Next run:

```bash
git bisect start
git bisect good <good tag>
git bisect bad <bad tag>
git bisect run ../bisect-run
```

I ran into a few gotchas that may be bugs that really need fixing. Occasionally
when running in '-noconsole' mode, I wouldn't see any prompt for a very long
time. When re-running with '-vga std', I'd see that it was waiting at GRUB. You
may need to modify timeouts such that you don't hit those issues without a VGA
console.

Overall, you can find these scripts [here][3].

Hopefully they will evolve a bit once its used more and more.

[1]: http://pad.lv/1100843
[2]: https://help.ubuntu.com/community/SerialConsoleHowto
[3]: https://github.com/arges/qemu-bisector

