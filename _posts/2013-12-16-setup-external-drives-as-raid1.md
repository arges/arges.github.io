---
layout: post
title: setup an external drives as raid1
date: '2013-12-16T14:37:00.000-08:00'
author: arges
tags:
- howto
- ubuntu
modified_time: '2014-03-05T14:36:04.242-08:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-1060336715553754712
blogger_orig_url: http://dinosaursareforever.blogspot.com/2013/12/setup-external-drives-as-raid1.html
---

I purchased a cheap two drive USB enclosure in order to setup an external drive
that had RAID-1 so I could backup photos and recordings.

First, I formatted both drives. Then I ran extended smart self-tests to ensure
I had decent drives. With RAID-1 and two drives I can only tolerate 1 drive
failure.

Next ensure mdadm is installed: `sudo apt-get install mdadm`

Determine which dev devices the disks show up as.  Next, create the raid device
pointing at the correct dev directory.

~~~bash
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdx
/dev/sdy sudo mkfs.ext /dev/md0
~~~

The device will start a resync process which on my system takes a really long
time (days). If you want to avoid this initial re-sync you can use
`--assume-clean` to avoid this. I would recommend letting it resync.

And there ya go.


