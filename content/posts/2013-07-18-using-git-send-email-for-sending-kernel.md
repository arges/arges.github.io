---
type: post
title: using git send-email for sending kernel patches
date: '2013-07-18T15:27:00.002-07:00'
author: arges
tags:
- git
- kernel
modified_time: '2013-09-10T09:32:47.926-07:00'
blogger_id: tag:blogger.com,1999:blog-7705678145617402978.post-6408714617303780462
blogger_orig_url: http://dinosaursareforever.blogspot.com/2013/07/using-git-send-email-for-sending-kernel.html
---

Sending patches to a Linux kernel mailing list can be done easily using
git-send-email. This post will help setup your environment and show you how to
format the git send-email command. In addition there are some more advanced
features that help when you need to send from multiple accounts.

### Git Setup

First, install git and git-email. Then, setup `~/.gitconfig` for your user
and proper sendemail section. This shows a sendemail setup for a typical single
gmail account.

~~~bash
[user]  
    name = Your Name
    email = user@gmail.com  
[sendemail]
    from = Your Name <user@gmail.com>
    smtpserver = smtp.gmail.com
    smtpuser = user@gmail.com
    smtpencryption = tls
    smtppass = PASSWORD
    chainreplyto = true
    smtpserverport = 587
~~~

However, it may be more useful to be able to easily send from multiple accounts.
This can be accomplished using the `--identify` flag in git.

~~~bash
[user]
    name = Your Name
    email =user@gmail.com
[user "work"]
    email = user@work.com
[user "gmail"]
    email = user@gmail.com
[sendemail "work"]
    from = Your Name <user@work.com>
    smtpserver = smtp.work.com
    smtpuser = me
    smtppass = PASSWORD
    smtpencryption = tls
    smtpserverport = 587
    chainreplyto = false
[sendemail "gmail"]
    from = Your Name <user@gmail.com>
    smtpserver = smtp.gmail.com
    smtpuser = user@gmail.com
    smtppass = PASSWORD
    smtpencryption = tls
    smtpserverport = 587
    chainreplyto = false
~~~

This way when you can select your identity to fill in these values. In addition
if you specify no identity you can have default fields if necessary. If you
added a [sendemail] field this would be called by default. Man git-config can
show you more [options][2].

### Formatting the Patch

Once we have the patch committed to the HEAD on our branch we format the patch
using: `git format-patch -1`.

This should produce a patch like `0001-blah.patch`
Then check for formatting errors using the checkpatch script provided in the
kernel repository: `./scripts/checkpatch.pl 0001-blah.patch`.

You should read the kernel [documentation][3] to get a better idea of what is
expected.

### Sending A Single Patch

Now we are ready to send a patch. The `./scripts/get_maintainer.pl` in the kernel
repository provides a way to specify whom needs to be CC'ed based on the
maintainers file, the history of the file, and which lines of code are changed.


~~~bash
git send-email --to <ml_list> --cc-cmd="scripts/get_maintainer.pl -i" \
  <0001-patch.patch>
~~~

The -i means the script is interactive and you can edit the list before sending.
Once you have completed the commands, the patch should be sent!

### Pull Requests

If you have your patches in a public git repository, it is sometimes easier to
send pull-requests for patches instead of sending.

~~~bash
git request-pull <hash right before your changes> \
    git://<public git repo> > request-pull.txt
~~~

Then add a 'Subject: ' line in the request-pull.txt that explains the pull
request. Adding --subject doesn't seem to work for me. In addition add any other
text.

~~~bash
git send-email --identity=gmail --to=<mailing list> \
    ./request-pull.txt
~~~

With this you should have a pull request sent!

[1]: http://git-scm.com/docs/git-send-email
[2]: http://git-scm.com/docs/git-config
[3]: https://www.kernel.org/doc/Documentation/SubmittingPatches
[4]: http://git-scm.com/docs/gitcredentials.html

