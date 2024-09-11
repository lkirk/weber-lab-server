# Server Setup

## Purpose

This document lays the steps to partition disks and install Ubuntu Server on the
lab server. In the unfortunate event where this all must be redone, this will
serve as a starting point for setting things up.

## Obtaining and Preparing the OS boot media

### Download Ubuntu 24.04 server installation software

Download the ubuntu install image
[here](https://ubuntu.com/download/server). For me it was the
ubuntu-24.04.1-live-server-amd64.iso file, but you might obtain a different
version.

To create the live boot media (on linux), I used the following command:
```bash
$ sudo dd bs=4M if=ubuntu-24.04.1-live-server-amd64.iso of=/dev/sda conv=fsync oflag=direct status=progress
```
NOTE: BE CAREFUL WITH THE `of` (out file). You must make sure you're not writing
to a drive that you are using on your computer. This must correspond to the
flash drive you've plugged into your computer. When in doubt, double and triple
check the device name (in my case `/dev/sda`). This command will delete any data
on the destination device. You can check for devices and where they're mounted
(tip: DON'T use mounted devices) with the `lsblk` command (linux only).

NOTE: the `dd` command may work on other unix-like systems (such as macos) by
omitting `status=progress`.


## Booting from flash drive

Insert the flash drive into the usb port on the front of the server and turn it
on. Hit `f12` repeatedly until the bios configuration screen appears.

On the lefthand side of the screen, you'll see a menu titled `UEFI Boot
Devices`. You will need to click on your USB disk to boot from it.

Once you get into the boot menu (It will say GNU GRUB at the top), you will need
to press `e`. You'll want to edit the boot options. There will be a line that
starts with `linux`. Directly at the end of the line (before the ---), add the
word `nomodeset`. This is necessary because the graphics card in the server does
not play well with linux for some reason. Once `nomodeset` is in place, press
`C-x` to boot.

You should now be in the Ubuntu Server Installer. It will prompt you for a
language choice.

## Installing Ubuntu Server

Select `English` Select `Identify Keyboard` Select `Ubuntu Server` (not
minimized). Also, select the `Search for third-party drivers` option.

At this point, one of the network devices should be autoconfigured. You should
see an IP address.

No proxy needed

Next, Ubuntu will update its package lists (it will say "mirror location passed
tests")

On the next screen, use the arrow keys to select `Custom storage layout`,
pressing the space bar to select.

<!-- Once in the partition editor, you'll want to go through each disk (All
prefixed --> <!-- with `WDC` or `Samsung`) and `Reformat` the existing ones. The
partition manager --> <!-- will also list your flash drive as an option, but
won't let you modify it. -->

<!-- Once all disks show up as `free space` (no partitions on them). Go ahead
and --> <!-- select one of the `Samsung` to `Use as boot device`. Next create a
1T partition --> <!-- on the boot disk with `ext4` format, selecting `Leave
unmounted` from the mount --> <!-- menu.. Next, create a partition on the boot
disk with the remaining size (should --> <!-- be ~`1.6T`), selecting `Leave
unmounted` from the mount menu. -->

<!-- Next, create an ext4 partition on the rest of the drives, leaving them
unmounted. -->

Next, we're going to drop into a shell to perform the partitioning (it's easier
this way, and the ubuntu install UI doesn't quite work). To do this, you'll want
to use the up arrow on the keyboard to select the Help menu in the top right
(it's white on yellow and very difficult to see). Select `Enter Shell`. You
should be dropped into a shell to run some commands.

First, we gather some information about what drives are present. We'll use
`lsblk --fs`, which outputs a bit more information about the filesystems,
including the disk labels, which can be helpful for deciding which disks to
partition. Currently, there are three spinning disks on this server and two NVME
drives. Spinning disks will show up as `sdX` (where X can be `a`, `b`, etc.) and
nvme drives will show up as `nvmesXnY` (where X and Y can be `1`, `2`,
etc.). For more information, see
[this](https://web.archive.org/web/20240511020657/https://www.debian.org/releases/bookworm/amd64/apcs04.en.html)
explanation of device naming and
[this](https://web.archive.org/web/20240827063550/https://utcc.utoronto.ca/~cks/space/blog/linux/NVMeDeviceNames)
explanation of nvme device naming. For more information on the design philosophy
used when devising our disk layout scheme, see TODO -- link.

Next, we execute a series of commands to create the partitions and
logical/virtual lvm partitions. Note, I've carefully selected the disks to run
these commands on.

First, just check that you've typed the device names in correctly. Note the
quotes. Quoting is smart, but not strictly necessary here.
```bash
# for d in sda sdb sdc nvme0n1 nvme1n1; do ls "/dev/$d"; done
```

Destroy the partition tables on all of our selected devices. I like
`sgdisk` because it's easily scriptable, but there's plenty of other options for
a more graphical experience (ie cfdisk)
```bash
# for d in sda sdb sdc nvme0n1 nvme1n1; do sgdisk -Z "/dev/$d"; done
```

Setup the gpt partition tables for the disks (just change the `-Z` to `-g`)
```bash
# for d in sda sdb sdc nvme0n1 nvme1n1; do sgdisk -g "/dev/$d"; done
```

Create the boot partition (quotes for safety)
```bash
# sgdisk -a 4096 -n '0:0:+1G' -t '0:EF00' -c '0:efi_boot' /dev/nvme0n1
``` This means that I'm creating a partition starting from 0 to 1G, aligned by
4096 bytes. The available partition types can be viewed with (`sgdisk -L`). I've
preselected EF00 (EFI system partition) and 8e00 (Linux LVM).

Create the system and scratch partition (same disk as previous step)
```bash
# sgdisk -a 4096 -n '0:0:+1T' -t '0:8e00' -c '0:ubuntu_system' /dev/nvme0n1
# sgdisk -a 4096 -n '0:0:0' -t '0:8e00' -c '0:scratch' /dev/nvme0n1
```

We're naming the partitions so that it's easy to identify their purpose, it has
no bearing on the outcome, they could be named whatever we want.

Partition the `work` disks:
```bash
# for d in sda sdb; do
      sgdisk -a 4096 -n '0:0:0' -t '0:8e00' -c '0:work' /dev/$d
  done
```

Partition the second `scratch` disk:
```bash
# sgdisk -a 4096 -n '0:0:0' -t '0:8e00' -c '0:scratch' /dev/nvme1n1
```

Partition the `snapshot` disk:
```bash
# sgdisk -a 4096 -n '0:0:0' -t '0:8e00' -c '0:snapshot' /dev/sdc
```

For good measure (to make sure the kernel has loaded the new partition tables):
```bash
# partprobe
```

Verify your work with `lsblk`. All of your created partitions should be present.

Create the physical volume groups that will get mounted as `/`, `/work`, and
`/scratch`, and `/snapshots`. Note that our device names now contain partition
identifiers. We're not touching the boot partition.
```bash
# for d in sda1 sdb1 sdc1 nvme0n1p2 nvme0n1p3 nvme1n1p1; do pvcreate "/dev/$d"; done
```

Verify with the `pvs` command.

Create the `ScratchVG` volume group, binding `nvme0n1p3` and `nvme1n1p1`.
```bash
# vgcreate ScratchVG /dev/nvme0n1p3 /dev/nvme1n1p1
```

Create the `WorkVG` volume group, binding `sda1` and `sdb1`.
```bash
# vgcreate WorkVG /dev/sda1 /dev/sdb1
```

Create the `SnapshotVG` volume group, binding `sda1` and `sdb1`.
```bash
# vgcreate SnapshotVG /dev/sdc1
``` 

Create the `SystemVG` volume group, binding `nvme0n1p2`.
```bash
# vgcreate SystemVG /dev/nvme0n1p2
```

Verify the results with the `pvs` and `pvdisplay` commands. Make sure that there
are no "new physical volumes". That means you've created a physical volume but
haven't added it to a volume group.

Create the swap logical volume before creating the rest (100%FREE will grab the
remaining space in the volume group). I'm creating a 64G swap -- this should be
sufficient. It can be resized later if need be.
```bash
# lvcreate -L 64G SystemVG -n SwapLV
```

Create the logical volumes for scratch and work
```bash
# for g in Scratch Snapshot System Work; do lvcreate -l '100%FREE' "${g}VG" -n "${g}LV"; done
```

Verify the results with the `lvs` and `lvdisplay` commands. In addition, run
`vgdisplay | grep -e Name -e Cur` to verify that every volume group (VG) has the
expected number of disks (PV, physical volumes), and that each volume group has
the expected number of logical volumes (SystemVG will have 2 -- one for the `/`
and one for the swap).

Verify our assumptions about naming before continuing
```bash
# for g in Scratch Snapshot System Work; do ls -la "/dev/mapper/${g}VG-${g}LV"; done
```
You should see the symlinks to `dm-0`, `dm-1`, etc...

Format all of the logical volumes with the ext4 filesystem
```bash
# for g in Scratch Snapshot System Work; do mkfs.ext4 "/dev/mapper/${g}VG-${g}LV"; done
```

Once this has been performed, exit back to the installer by pressing 'C-d'.

<!-- Make the swap partition -->
<!-- ```bash -->
<!-- # mkswap /dev/mapper/SystemVG-SwapLV -->
<!-- ``` -->

<!-- Make the directories to serve as mount points -->
<!-- ```bash -->
<!-- mkdir /scratch /work /snapshot -->
<!-- ``` -->

<!-- Mount all of the logical volumes -->
<!-- ```bash -->
<!-- mount /dev/mapper/ScratchVG-ScratchLV /scratch -->
<!-- mount /dev/mapper/WorkVG-WorkLV /work -->
<!-- mount /dev/mapper/SnapshotVG-SnapshotLV /snapshot -->
<!-- mount /dev/mapper/SystemVG-SystemLV / -->
<!-- ``` -->

<!-- Activate the swap partition -->
<!-- ```bash -->
<!-- # swapon /dev/mapper/SystemVG-SwapLV -->
<!-- ``` -->

<!-- Verify your work with `lsblk`. You should see all of your volume groups/logical -->
<!-- volumes mounted to their corresponding directories. -->

To get the system installer to detect the work done in the terminal, you'll want
to select `[ Back ]` to jump to the previous step, then `[ Done ]` to jump back
to the partitioning step.

All of the changes should be detected. Next, select the mount points for the
logical volumes (LV). Use the up/down arrows to select each LV, hit `Enter` and
select `Edit`. Leave the format, but change the mount point for each LV.

| Logical Volume | Mountpoint |
|:--|:--|
| ScratchLV | /scratch |
| WorkLV | /work |
| SnapshotLV | /snapshots |
| SystemLV | /system  |

For the swap, select the `Format` as `swap`

NOTE: for the / /snapshots /work and /scratch mountpoints, you'll have to select
`Other` as the mountpoint and manually type the mountpoint. It will warn you
about mounting `/`, but ignore it.

There should be one remaining partition shown (except for the partitions it
shows from your bootable install flash drive). You'll want to select the disk
(in this case, it's the `Samsung_SSD_*`) by pressing `Enter` and selecting `Use
As Boot Device`. The final partition should disappear.

Next comes basic server configuration

The server's name will end up being the `hostname`. I set it to
`stickleback`. Setup your name/user/password as desired.

Do not enable `Ubuntu Pro`, it is a paid service.

Select the `Install OpenSSH server`. No need to import keys at the moment.

I opted to install `canonical-livepatch` (which will enable kernel updates
without the need to reboot).

The install should have completed successfully. Follow the instructions to reboot.

After rebooting, you'll need to boot with `nomodeset` as we did before. Except
that the command you're editing will be more complicated. At the end of the line
beginning with `linux` add ` nomodeset` (note the additional space).

Next, we edit the `grub` config so that we always boot with `nomodeset`. We're
going to edit the `/etc/default/grub` file. I use `vim`, but feel free to use
any text editor you desire.

```bash
sudo vim /etc/default/grub
```

Change `GRUB_CMDLINE_LINUX_DEFAULT=""` to `GRUB_CMDLINE_LINUX_DEFAULT="nomodeset"`

Then run

```bash
sudo update-grub
```

And reboot. If this worked, you should be able to boot without any manual
intervention. This can be done with the `reboot` command.

After rebooting, I went ahead and updated all packages
```bash
sudo apt update
sudo apt upgrade
```

By default, the server does not come with any graphical interface, I installed
one so that management tasks are a bit easier. I chose xfce4 because it's
lightweight and simple to use. It can be deleted at any point. In addition, I
installed zsh, firefox, and all of the software building tools `build-essential`

```
sudo apt install xfce4 zsh firefox build-essential
```

To globally configure zsh, I added the grml debian package repository to the
system's repos (show where that is... commands used).
[https://deb.grml.org/](https://deb.grml.org/)
configuration. This provides many nice defaults for tab completion, etc.

grml zsh config

```
wget https://deb.grml.org/repo-key.gpg

sudo cp repo-key.gpg  /usr/share/keyrings/grml.gpg

echo 'Types: deb
URIs: http://deb.grml.org/
Suites: grml-stable
Components: main
Signed-By: /usr/share/keyrings/grml.gpg' | sudo tee /etc/apt/sources.list.d/grml.sources

sudo apt update

sudo apt install grml-etc-core

chsh -s /usr/bin/zsh
```

<!-- Once in place I ran -->
<!-- ``` -->
<!-- sudo apt update -->
<!-- sudo apt install grml-etc-core -->
<!-- ``` -->

<!-- Change default shell for the current user: -->
<!-- ``` -->
<!-- chsh -s /usr/bin/zsh -->
<!-- ``` -->

rstudio

https://github.com/rstudio/rstudio/issues/14757
https://github.com/rstudio/rstudio/issues/14336
https://dailies.rstudio.com/rstudio/cranberry-hibiscus/server/noble-amd64
https://s3.amazonaws.com/rstudio-ide-build/server/jammy/amd64/rstudio-server-2024.09.0-360-amd64.deb


```
wget https://s3.amazonaws.com/rstudio-ide-build/server/jammy/amd64/rstudio-server-2024.09.0-360-amd64.deb
sudo apt install ./rstudio-server-2024.09.0-360-amd64.deb
sudo systemctl --status rstudio-server.service
sudo systemctl status rstudio-server.service
```

```
sudo wget https://raw.githubusercontent.com/conda-incubator/conda-zsh-completion/main/_conda -O /usr/share/zsh/functions/Completion/Linux/_conda
```

```
sudo useradd --system conda
sudo chsh -s /bin/bash conda
sudo mkdir /home/conda
sudo chown conda:conda /home/conda
echo '[ -f /opt/conda/etc/profile.d/conda.sh ] && source /opt/conda/etc/profile.d/conda.sh' | sudo -u conda tee /home/conda/.bashrc
echo '[ -f /opt/conda/etc/profile.d/conda.sh ] && source /opt/conda/etc/profile.d/conda.sh' | sudo -u conda tee /home/conda/.bash_profile
```

https://docs.anaconda.com/miniconda/miniconda-install/

```
sudo mkdir /opt/conda
sudo chown conda:conda /opt/conda
sudo su conda
cd /opt/conda
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod +x Miniconda3-latest-Linux-x86_64.sh
./Miniconda3-latest-Linux-x86_64.sh -b -f -p /opt/conda
rm Miniconda3-latest-Linux-x86_64.sh
conda config --prepend channels conda-forge
conda config --prepend channels bioconda
conda install conda-protect
conda protect base
```
https://github.com/conda-incubator/conda-protect


```
sudo groupadd admin
sudo usermod -G admin weber
```

TODO: conda sudo config
