---
type: post
title: using gcov/lcov with the linux kernel
date: '2014-11-13T09:45:00.003-08:00'
author: arges
tags:
- tools
- linux
- testing
modified_time: '2014-11-13T09:45:58.436-08:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-8591994176648230821
blogger_orig_url: http://dinosaursareforever.blogspot.com/2014/11/using-gcovlcov-with-linux-kernel.html
---

GCOV/LCOV are amazing tools to figure out code coverage in the Linux kernel. It
is fairly trivial to setup on your own machine. First, enable the following in
your kernel configuration:

~~~bash
CONFIG_GCOV_KERNEL=y
CONFIG_GCOV_PROFILE_ALL=y
CONFIG_GCOV_FORMAT_AUTODETECT=y
~~~

Build the kernel and install headers/debug/images to your machine. Note there
may be issues if you run this on a separate machine so consult the official
[documentation][1] for additional information.

Boot the machine with the instrumented kernel and use the following to verify
GCOV_KERNEL was setup properly:

~~~bash
sudo apt-get install lcov gcc
SRC_PATH=~/src/linux # or whatever your path to your kernel tree is
gcov ${SRC_PATH}/kernel/sched/core.c -o /sys/kernel/debug/gcov/${SRC_PATH}/kernel/sched/
~~~

Obviously this would be useful for quick tests, but perhaps we want to test
coverage after a test, and display results graphically. We can reset counters
and use [lcov][2] to accomplish this:

~~~bash
# reset counters
sudo lcov --zerocounters
# run test
./test.sh
# generate coverage info and generate webpage
sudo lcov -c -o kerneltest.info
sudo genhtml -o /var/www/html kerneltest.info
~~~

[1]: https://www.kernel.org/doc/Documentation/gcov.txt "gcov docs"
[2]: http://ltp.sourceforge.net/documentation/how-to/UsingCodeCoverage.pdf "lcov docs"

