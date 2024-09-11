# Disk Layout

This document documents the design decisions and general philosophy used to
create the filesystem layout for the server.

## Layout

With the current disk configuration, we plan to create 4 mounts from the 5 disks
present.

| Device | Number of partitions | Filesystem Type | Boot disk? |
|:--|:--|:--|
| /dev/sda     | 1 | ext4 | N |
| /dev/sdb     | 1 | ext4 | N |
| /dev/sdc     | 1 | ext4 | N |
| /dev/nvme0n1 | 3 | ext4 | Y |
| /dev/nvme1n1 | 1 | ext4 | N |

Why this configuration? The boot disk needs too have a specific boot partition
to function. We're also cutting the boot disk into pieces because 4T is too
large for a system install (it's a waste of storage space).


## Why LVM?



## More on LVM

There are generally excellent resources on the archlinux wiki:
[LVM](https://wiki.archlinux.org/title/LVM), [Installing on
LVM](https://wiki.archlinux.org/title/Install_Arch_Linux_on_LVM) 


## Why not X?

BTRFS

ZFS

RAID
