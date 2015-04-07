---
layout: post
title: using kgraft with ubuntu
date: '2014-06-06T11:50:00.000-07:00'
author: arges
tags:
- kgraft
- kernel
- ubuntu
modified_time: '2014-06-06T11:50:57.621-07:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-2833524714529676554
blogger_orig_url: http://dinosaursareforever.blogspot.com/2014/06/using-kgraft-with-ubuntu.html
---

New live kernel patching projects have hit LKML recently [here][1] and
[here][2], and I've taken the opportunity to test drive kGraft with the Ubuntu
kernel. This post documents how to get a sample patch working.

First, I had to take the [patches][3] and apply them against the
ubuntu-utopic kernel, which is based on 3.15-rc8 as of this post. They
cherry-picked cleanly and the branch I'm using is stored [here][4]. In addition
to applying the patches I had to also enable CONFIG_KGRAFT. A pre-built test
kernel can be downloaded [here][5].

Next, I created a test VM and installed the test kernel, headers, and build
dependencies into that VM and rebooted. Now after a successful reboot, we need
to produce an actual patch to test. I've created a github [project][6] with the
sample patch; to make it easy to clone and get started.

```bash
sudo apt-get install git build-essential
git clone https://github.com/arges/kgraft-examples.git
cd kgraft-examples
make
```

The code in kgraft_patcher.c is the example found in [samples/kgraft][7]. Now we
can build it easily using the Makefile I have in my project by typing make.

Next, the module needs to be inserted using the following:
```sudo insmod ./kgraft_patcher.ko```

Run the following to see if the module loaded properly:
```lsmod | grep kgraft```

You'll notice some messages printed with the following:

```
[  211.762563] kgraft_patcher: module verification failed: signature and/or
required key missing - tainting kernel
[  216.800080] kgr failed after timeout (30), still in degraded mode
[  246.880146] kgr failed after timeout (30), still in degraded mode
[  276.960211] kgr failed after timeout (30), still in degraded mode
```

This means that not all processes have entered the kernel and may not have a
"new universe" flag set.  Run the following to see which processes still needs
to be updated.
```cat /proc/*/kgr_in_progress```

In order to get all processes to enter the kernel sometimes a signal needs to be
sent to get the process to enter the kernel.

An example of this is found in the kgraft-examples called [hurryup.sh][6]:

```bash
#!/bin/bash
for p in $(ls /proc/ | grep '^[0-9]'); do
  if [[ -e /proc/$p/kgr_in_progress ]]; then
    if [[ `sudo cat /proc/$p/kgr_in_progress` -eq 1 ]]; then
     echo $p;
     sudo kill -SIGCONT $p
    fi
  fi
done
```

Here is checks for all processes that have 'kgr_in_progress' set and sends a
SIGCONT signal to that process. 

I've noticed that I had to also send a SIGSTOP followed by a SIGCONT to finally
get everything synced up.

Eventually you'll see:
```[ 1600.480233] kgr succeeded```

Now your kernel is running the new patch without rebooting!

[1]: https://lkml.org/lkml/2014/4/30/477
[2]: https://lkml.org/lkml/2014/5/1/273
[3]: https://git.kernel.org/cgit/linux/kernel/git/jirislaby/kgraft.git/
[4]: http://zinc.ubuntu.com/git?p=arges/ubuntu-utopic.git;a=shortlog;h=refs/heads/kgraft-utopic
[5]: http://people.canonical.com/~arges/kgraft-utopic/
[6]: https://github.com/arges/kgraft-examples
[7]: https://git.kernel.org/cgit/linux/kernel/git/jirislaby/kgraft.git/tree/samples/kgraft/kgraft_patcher.c?h=kgraft
[8]: https://git.kernel.org/cgit/linux/kernel/git/jirislaby/kgraft.git/tree/tools/kgraft/create-stub.sh?h=kgraft


