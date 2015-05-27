---
layout: post
title: accessing maas database directly
date: '2015-05-27T13:02:46-0500'
author: arges
tags:
- linux
- ubuntu
- maas
modified_time: '2015-05-27T13:02:52-0500'

---

Let's say you've messed up your MAAS installation and have no idea how to
recover data about your nodes. Have no fear, you can access the django managed
database directly.

Just use the following as root:

```
maas-region-admin dbshell --installed
```

Now you have access to an SQL command line interface.

For example if I want to see all hostnames and power_parameters for my servers,
I can do the following:

```
select hostname,power_parameters from maasserver_node;
```

