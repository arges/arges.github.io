---
type: post
title: grub-reboot with submenus
date: '2016-02-25T10:05:51-0500'
author: arges
tags:
- ubuntu
- grub
modified_time: '2016-02-25T10:05:56-0500'
---

Occasionally it is useful to be able to reboot a machine into an earlier kernel
and watching and waiting for the grub menu isn't always feasible or convenient.

The command `grub-reboot` allows you to set a temporary entry to use for the
next reboot; however you need to know the correct input in order for it to work.

First, look at `/boot/grub/grub.cfg` and determine which kernel you want to
reboot into. Note the menu and submenus to construct a string of the form
`menu>submenu`. For example if you want to reboot into an older kernel you can
do:

~~~bash
sudo grub-reboot "Advanced options for Ubuntu>Ubuntu, with Linux 4.2.0-16-generic"
sudo reboot
~~~

After rebooting you'll be in the 4.2.0-16-generic kernel!
