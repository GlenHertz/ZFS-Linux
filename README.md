ZFS-Linux
=========

ZFS on Ubuntu 14.04 Trusty Tahr (Linux Mint 17)

# Background

I want to setup a computer to be resistant to hard drive failures and provide easy snapshoting and backups.  I chose ZFS for these reasons, as well as it being very easy to adminster.

For hardware I have a motherboard with two 6 Gbps SATA ports and two 3 Gbps SATA ports.  This is a bit limiting as a raidz1 partition takes at least 3 drives.  I also want to be able to have a fast root partition on a SSD.  So I will setup Linux Mint 17 with the following configuration:


* 60 GB SSD drive (using 6 Gbps SATA, port 0)
  * Part 1: 256 Boot disk, /boot, ext2
  * Part 2: 1.4 GB recovery disk (using http://people.debian.org/~jgoerzen/rescue-zfs/)
  * Part 3: 20 GB disk, /, ZFS mirror
  * Part 4: 20 GB disk, /, ZFS mirror
  * Part 5: 18.5 GB disk, ZFS cache

This isn't ideal since the ZFS mirror is on the same controller and same drive.  They really should be on separate SATA ports and separate drives so this will only protect me from bits on the drive dying but it gives me a bit more security and I don't care about the OS partition that much.  I am more interesting in being able to do ZFS snapshots

* 3TB WD NAS drives (using one 6 Gbps and two 3 Gbps SATA ports)
  * Complete drives, in ZFS raidz1 pool
  * Mount points as /zfs, /zfs/home -> /home (compression), /zfs/mythtv -> /mythtv (no compression)

The ZFS pool should provide about 2TB of space.  One drive can fail and it will be self healing.  

A USB drive is very useful, lets create a bootable USB drive with System Rescue CD with ZFS and Linux Mint 17:

* Download the rescue CD (http://people.debian.org/~jgoerzen/rescue-zfs/)
* Download Linux Mint 17 (must be 64-bit for ZFS)
* Install a USB boot disk creator 
  * On Ubuntu I used unetbootin (USB statup creator only works for Ubuntu ISOs) which required me to also install the extlinux package.
  * Then launched unetbootin and installed System Recovery ISO onto first partition and Linux Mint 17 on second partition.

 
# First boot-up

* Boot from Linux Mint 17 live CD
* Get the ZFS PPA: `Linux Mint Menu >> Software Sources >> PPAs >> ppa:zfs-native/stable`
* Install `wajig`, a better `apt-get` wrapper and install ZFS kernel module:
 
```
sudo -i
apt-get install wajig
wajig update
wajig install build-essential
wajig install spl-dkms zfs-dkms ubuntu-zfs mountall
modprobe zfs
dmesg | grep ZFS:
# if successful, should return: ZFS: Loaded module v0.6.3-2~trusty, ZFS pool version 5000, ZFS filesystem version 5
```

If that works, now we can create ZFS volumes, etc.

# Partitioning

**First backup your data!  You have been warned!**

Using `cfdisk` from a Terminal, partition the SSD drive:

* Part 1: primary, size=256 MB, type=83 (ext2), bootable 
* Part 2: primary, size=20000 MB, type=BF (Solaris)
* Part 3: primary, size=20000 MB, type=BF (Solaris)
* Part 3: primary, size=remainder, type=BF (Solaris)

When finished it should look something like this, `fdisk -l /dev/sda`:

```
   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048      499711      248832   83  Linux
/dev/sda2          499712    39561664    19530976+  bf  Solaris
/dev/sda3        39561665    78623617    19530976+  bf  Solaris
/dev/sda4        78623618   117231407    19303895   bf  Solaris
```

Now look at the partition descriptors to verify with `ls -l /dev/disk/by-id`.  You should see the name of the drive and symlinks to /dev/sda:

```
mint ~ # ls -l /dev/disk/by-id/
total 0
lrwxrwxrwx 1 root root  9 Jun 14 14:56 ata-KINGSTON -> ../../sda
lrwxrwxrwx 1 root root 10 Jun 14 14:56 ata-KINGSTON-part1 -> ../../sda1
lrwxrwxrwx 1 root root 10 Jun 14 14:56 ata-KINGSTON-part2 -> ../../sda2
lrwxrwxrwx 1 root root 10 Jun 14 14:56 ata-KINGSTON-part3 -> ../../sda3
lrwxrwxrwx 1 root root 10 Jun 14 14:56 ata-KINGSTON-part4 -> ../../sda4
```

With ZFS you should always use the above names (instead of `/dev/sda2`) since they are always the same when adding/removing drives and reboots.

The other 3 large hard drives don't need to be formated or partitioned since they will be completely **deleted** and used entirely for ZFS.  To add to ZFS first get their id from the filesystem:

```
mint ~ # ls -l /dev/disk/by-id/ | grep -v part | grep ata-
lrwxrwxrwx 1 root root  9 Jun 14 14:56 ata-KINGSTON -> ../../sda
lrwxrwxrwx 1 root root  9 Jun 14 15:16 ata-WDC_WD30EFRX-1 -> ../../sdc
lrwxrwxrwx 1 root root  9 Jun 14 15:16 ata-WDC_WD30EFRX-2 -> ../../sdd
lrwxrwxrwx 1 root root  9 Jun 14 15:16 ata-WDC_WD30EFRX-3 -> ../../sdb
```

The part2 and part3 partitions on the SSD are a mirror.  The three large hard drives create a raidz1 array and the SSD part4 is the cache for the slow hard drives (making them into a fast hybrid drive).

```
zpool create -f zfsraid raidz1 ata-WDC_WD30EFRX-1 ata-WDC_WD30EFRX-2 ata-WDC_WD30EFRX-3 cache ata-KINGSTON-part4
zpool create zfsroot mirror ata-KINGSTON-part2 ata-KINGSTON-part3
zpool list -v
NAME   SIZE  ALLOC   FREE    CAP  DEDUP  HEALTH  ALTROOT
zfsraid  8.12T  1.13M  8.12T     0%  1.00x  ONLINE  -
  raidz1  8.12T  1.13M  8.12T         -
    ata-WDC_WD30EFRX-1      -      -      -         -
    ata-WDC_WD30EFRX-2      -      -      -         -
    ata-WDC_WD30EFRX-3      -      -      -         -
cache      -      -      -      -      -      -
  ata-KINGSTON-part4  18.4G    26K  18.4G         -
zfsroot  18.5G   126K  18.5G     0%  1.00x  ONLINE  -
  mirror  18.5G   126K  18.5G         -
    ata-KINGSTON-part2      -      -      -         -
    ata-KINGSTON-part3      -      -      -         -
```

