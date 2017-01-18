---
layout: post
title: rebooting in ansible
date: '2017-01-18T08:52:35-0600'
author: arges
tags:
- ansible
- linux

---

Ansible does many things fairly well, but rebooting a remote machine seems to
be tricky. Personally, it seems like a plugin would serve well here, but for
now everyone needs to write their own tasks.

Here's what I finally got working:
~~~yml
- shell: nohup bash -c 'sleep 2 && shutdown -r now' &
  async: 0
  poll: 0
  ignore_errors: true

- local_action: wait_for host={{ ansible_host }} search_regex='OpenSSH' port={{ ansible_port }} state='started' delay=15 timeout=300
  become: no
~~~
