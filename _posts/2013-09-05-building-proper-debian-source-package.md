---
layout: post
title: building a proper debian source package with dkms
date: '2013-09-05T07:10:00.001-07:00'
author: arges
tags:
- debian
- dkms
- ubuntu
modified_time: '2013-09-10T09:31:59.572-07:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-5221954440630906155
blogger_orig_url: http://dinosaursareforever.blogspot.com/2013/09/building-proper-debian-source-package.html
---

How can one use DKMS to build a proper debian source package? The ~~~mkdsc~~~
command will actually generate one automatically, but there are a few more
steps to simplify it, and bring it up to date.

First follow the steps [here][1] for setting up a DKMS package. Make sure you
can build it using dkms. Then do the following:

~~~bash
#First pull in some dependencies as necessary
sudo apt-get install devscripts debhelper

# Create the debian source package
sudo dkms mkdsc -m hello -v 0.1

# Create a directory to work in, and let’s copy those files into it
mkdir ~/dsc && cd ~/dsc
cp /var/lib/dkms/hello/0.1/dsc/* .

# Extract the .dsc to be able to edit the directory
dpkg-source -x hello-dkms_0.1.dsc 
cd hello-dkms-0.1
~~~

If we run debuild -uc -us, we see a few lintian errors and warnings:

~~~bash
W: hello-dkms source: package-file-is-executable debian/changelog
W: hello-dkms source: package-file-is-executable debian/control
W: hello-dkms source: package-file-is-executable debian/copyright
W: hello-dkms source: package-file-is-executable debian/dirs
E: hello-dkms source: no-human-maintainers
W: hello-dkms source: debian-rules-ignores-make-clean-error line 28
W: hello-dkms source: debian-rules-missing-recommended-target build-arch
W: hello-dkms source: debian-rules-missing-recommended-target build-indep
W: hello-dkms source: ancient-standards-version 3.8.1 (current is 3.9.3)
E: hello-dkms: no-copyright-file
E: hello-dkms: extended-description-is-empty
W: hello-dkms: non-standard-dir-perm usr/src/hello-0.1/ 0655 != 0755
~~~

The errors are normal, and if this was a real package those will be taken care
of since that information will need to be filled in anyway.

To address the executable issues, just chmod -x those files:
~~~chmod -x debian/co* debian/dirs debian/ch*~~~.

To address some of the other issues, we can just completely modify and change
the rules file. Because debhelper has a helper for DKMS specifically we should
use it.

In addition, if we do something like dch -i you’ll notice some errors, since
our source directory is hardcoded to hello-0.1. So we can modify it to be in a
~~~src~~~ directory and get around this:
~~~mv hello-0.1 src~~~

Here is how I modified my ~~~debian/rules~~~ file:

~~~bash
#!/usr/bin/make -f
# -*- makefile -*-

# Uncomment this to turn on verbose mode.
#export DH_VERBOSE=1

NAME=hello
DEB_NAME=$(NAME)-dkms
VERSION=$(shell dpkg-parsechangelog |grep ^Version:|cut -d ' ' -f 2)

%:
        dh $@ --with dkms

override_dh_install:
        dh_install src/ usr/src/$(NAME)-$(VERSION)
        find "debian/$(DEB_NAME)/usr/src/$(NAME)-$(VERSION)" -type f -exec chmod 644 {} \;

override_dh_dkms:
        dh_dkms -V $(VERSION)

override_dh_auto_build:
override_dh_auto_install:
~~~

Now there are some things that can be removed from the package completely:
~~~rm common.postinst Makefile~~~

Now to update the control file to use modern versions, a proper description,
and make yourself a maintainer!

~~~bash
Source: hello-dkms
Section: misc
Priority: optional
Maintainer: Dude Bodacious <dude@awesomeradical.com>
Build-Depends: debhelper (>= 8), dkms
Standards-Version: 3.9.3

Package: hello-dkms
Architecture: all
Depends: dkms, ${misc:Depends}
Description: hello driver in DKMS format.
A completely useful kernel module.
~~~

Now debuild -uc -us and fix remaining issues.

[1]: https://wiki.ubuntu.com/Kernel/Dev/DKMSPackaging
[2]: https://help.ubuntu.com/community/Kernel/DkmsDriverPackage
[3]: http://basilevsthecat.blogspot.com/2011/11/how-to-build-dkms-debian-package.html


