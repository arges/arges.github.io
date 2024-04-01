---
type: post
title: linux make deb-pkg speedup
date: '2015-07-30T15:20:31-0500'
author: arges
tags:
- linux
- ubuntu
- debug
modified_time: '2015-07-30T15:20:37-0500'

---

Because I've run `make deb-pkg` so many times, I've started to see exactly where
it starts to slow down even with really large machines. Observing cpu usage, I
noticed that many parts of the build were serialized on a single core. Upon
further investigation I found the following.

### Upstream Packaging

Module installation takes a really long time, and when building with
`make deb-pkg -jN`, you'll see output like the following:

~~~
make[2]: warning: jobserver unavailable: using -j1.  Add `+' to parent make rule.
~~~

The main issue here is what we're calling `$MAKE` from inside a bash script and
it no longer has its jobserver argument. In order to fix this we need to ensure
that the parent Makefile calls the script with the '+' prepended to the call.

Next, this particular for loop in `scripts/package/builddep` causes making debug
packages take forever:

~~~bash
for module in $(find $tmpdir/lib/modules/ -name *.ko -printf '%P\n'); do
	module=lib/modules/$module                              
	mkdir -p $(dirname $dbg_dir/usr/lib/debug/$module)      
	# only keep debug symbols in the debug file             
	$OBJCOPY --only-keep-debug $tmpdir/$module $dbg_dir/usr/lib/debug/$module
	# strip original module from debug symbols              
	$OBJCOPY --strip-debug $tmpdir/$module                  
	# then add a link to those                              
	$OBJCOPY --add-gnu-debuglink=$dbg_dir/usr/lib/debug/$module $tmpdir/$module
done   
~~~

We can use the xargs paradigm to make it much quicker:

~~~bash
find $tmpdir/lib/modules/ -name *.ko -printf '%P\n' | xargs -I {} sh -c '
	mkdir -p $(dirname '"$dbg_dir"'/usr/lib/debug/lib/modules/$1);'"
	# only keep debug symbols in the debug file
	$OBJCOPY --only-keep-debug $tmpdir/lib/modules/{} \
		$dbg_dir/usr/lib/debug/lib/modules/{};
	# strip original module from debug symbols
	$OBJCOPY --strip-debug $tmpdir/lib/modules/{};
	# then add a link to those
	$OBJCOPY --add-gnu-debuglink=$dbg_dir/usr/lib/debug/lib/modules/{} \
		$tmpdir/lib/modules/{};
" -- {}
~~~

### Results

Using a distro config (from Ubuntu) and profiling the following after running
make clean:

~~~
time make deb-pkg -j`nproc`
~~~

Results without the patches:

~~~
real    41m39.846s 
user    179m10.908s
sys     138m4.660s 

real    40m56.240s 
user    182m17.292s
sys     140m18.812s

~~~

Results with the patches:

~~~
real    35m27.924s     
user    181m45.868s    
sys     148m55.620s    

real    36m22.633s 
user    182m28.028s
sys     148m17.724s

~~~

A speedup of 5m isn't that bad, and keep in mind much of this is the actual
compile too.

Patches have been posted [here][1].

Part of the patch has been merged [here][2].

[1]: https://lkml.org/lkml/2015/4/24/619
[2]: http://www.spinics.net/lists/linux-kbuild/msg11317.html
