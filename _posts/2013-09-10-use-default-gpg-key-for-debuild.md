---
layout: post
title: use default gpg key for debuild
date: '2013-09-10T09:30:00.002-07:00'
author: arges
tags:
- debian
- ubuntu
modified_time: '2014-03-05T14:33:50.552-08:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-8298458521934635665
blogger_orig_url: http://dinosaursareforever.blogspot.com/2013/09/use-default-gpg-key-for-debuild.html
---

When using `debuild -S` to build a package on new machines, I need to go
through the process of ensuring that my gpg is set up properly. If set up
incorrectly, I get the following message with `debuild -S`:

~~~bash
gpg: skipped "User <user@host.com>": secret key not available
~~~

To fix this, I go through the following steps:

1. Ensure I have proper gpg keys set up. You can check if yours is installed
properly using: `gpg --list-keys`

2. Add `DEBEMAIL` and `DEBFULLNAME` to `~/.bashrc`. Ensure
`DEBEMAIL` matches the email in the matching gpg key, and ensure
DEBFULLNAME matches the name in same key.  Put these into your ` ~/.bashrc`
file or wherever they will be sourced into your environment.

3. Make sure your key is the default key: Edit `~/.gnupg/gpg.conf` such
that default-key is set to the proper key. This should match the 8 character
fingerprint that you verified in (1).

4. Make sure the same key is the default for debuilder: Edit
/etc/devscripts.conf with the folllowing: `DEBSIGN_KEYID=XXXXXX` where
`XXXXXX` is your key.

5. Now next time you use `debuild -S` it should use the proper gpg key,
name and email address!


