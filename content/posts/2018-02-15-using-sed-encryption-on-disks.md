---
type: post
title: using sed encryption on harddrives
date: '2018-02-15T21:42:03-0600'
author: arges
tags:
- linux
- hardware
modified_time: '2018-02-16T21:30:24-0600'

---

Disk encryption can be accomplished on Linux in many ways depending on your
hardware. [LUKS][1] provides block-layer level encryption, and most modern
Linux distros provide easy ways to use and setup keys with the device. The
[eCryptFS][2] project provides filesystem level encryption which gives a bit
more flexibility.

However, many new disks have encryption built-in to the drive's controller and
can be interfaced with utilities such as [hdparm][3] and [sedutil-cli][4].
This built in encryption is called FDE (Full Disk Encryption) or SED (Self
Encrypting Drives). When these disks have encryption keys enabled, upon power
loss they become locked and must be unlocked before they are used. Different
protocols can be used for disks, here I'll cover OPAL and ATA type commands.

OPAL SED Drives
===============

For OPAL drives you can use sedutil-cli to scan for drives:
```
sedutil-cli --scan
```

Here you can determine if devices you have are capable. If you have devices
behind a hardware RAID controller you may need to access the device via the
scsi_generic layer via `/dev/sgX`.


First you need to initially setup the drive using this command:
```
sedutil-cli --initialsetup <password> <device>
```

Once you set the password you can lock the drive:
```
sedutil-cli --enableLockingRange 0 <password> <device>
```

At this point if you remove power from the device you will need to unlock it.
To unlock the drive:
```
sedutil-cli --setLockingRange 0 rw <password> <device>
```

Or disable drive locking:
```
sedutil-cli --disableLockingRange 0 <password> <device>
```

For OPAL drives there are multiple password users: SID and Admin1. Admin1 can
manage locking and SID can manage everything. When you initialize with
--initialsetup both SID and Admin1 passwords are set to the same thing.

You can also change the individual passwords using the following:
```
sedutil-cli --setSIDPassword <SIDpassword> <newSIDpassword> <device>
sedutil-cli --setAdmin1Pwd <Admin1password> <newAdmin1password> <device>
```

Secure erase can be accomplished using the SID password if you know it:
```
seduitl-cli --revertTPer <SIDpassword> <device>
```

If you no longer have the password you'll need to physically look at the drive
and get the PSID. This will be on the sticker on the drive and some even have
QR codes with this information on the drive. To revert with the PSID use:
```
sedutil-cli --yesIreallywanttoERASEALLmydatausingthePSID <PSIDALLCAPSNODASHED> <device>
```

ATA SED Drives
==============

Some devices such as INTEL, Micron, or Samsung SSDs with SED encryption can
only be driven via an ATA interface and do not implement the OPAL interface.

To query the device use the standard:
```
hdparm -I /dev/sgX
```

Here you'll see a section as follows from the output:
```
Security:
	Master password revision code = 65534
		supported
	not	enabled
	not	locked
		frozen
	not	expired: security count
		supported: enhanced erase
	2min for SECURITY ERASE UNIT. 8min for ENHANCED SECURITY ERASE UNIT.
```

If you see `supported` you can use hdparm to enable locking the drives.
Other fields of note are `enabled` which indicates if a password is set or not.
The `locked` field indicates if the drive is locked or unlocked.

There are two passwords 'user' and 'master' which can have different
capabilities depending on how things are set up. The flag `--user-master` can
specify 'u' for user or 'm' for master.

To set a password for the `user` account:
```
hdparm --user-master u --security-set-pass <password> /dev/sgX
```

Locking the drive  will happen when you completely remove power from the disk.
To unlock the drive you can use:
```
hdparm --user-master u --security-unlock <password> /dev/sgX
```

To disable the password:
```
hdparm --user-master u --security-disable <password> /dev/sgX
```

To [secure erase][5] the drive first ensure drive is not frozen and security is
enabled then run:
```
hdparm --user-master u --security-erase <password> /dev/sgX
```

[1]: https://gitlab.com/cryptsetup/cryptsetup/
[2]: http://ecryptfs.org/
[3]: https://wiki.archlinux.org/index.php/hdparm
[4]: https://github.com/Drive-Trust-Alliance/sedutil
[5]: https://ata.wiki.kernel.org/index.php/ATA_Secure_Erase
