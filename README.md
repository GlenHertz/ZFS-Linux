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

 



