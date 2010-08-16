---
title: Increasing hard disk size with LVM on VMWare Fusion Linux guest
---

I currently use VMware Fusion to maintain and run a Linux guest on my Macbook.
When I originally installed the guest, I assumed that a 20 GB virtual disk
would be more than enough space for my Linux hacking.

As you can probably infer from the title of this post, some recent developments
invalidated that assumption and necessitated more disk space. What follows is a
procedure I cobbled together from a bunch of disparate sources that works for my
particular setup (VMware Fusion 2.0.4, Fedora 10 guest w/LVM and an ext3
filesystem). These instructions involve modifying physical partitions, logical
volumes, and your filesystem so I advise you to read through them to the end
before getting started.

The very first step is to use Fusion to increase the "physical size" of the
virtual disk. This functionality is accessible through the _Virtual Machine
-> Hard Disk -> Hard Disk Settings_ menu item. The resulting dialog contains a
_Disk size_ slider. Note that this slider is not accessible unless the VM in
question is shut down and (more annoyingly) all pre-existing snapshots are deleted.
I just had one snapshot to delete, but it took a while, so be patient. After
you've increased the disk size, probably wouldn't hurt to take a new snapshot,
just in case something goes wrong with the procedure below.

Now that we've increased the disk size (I increased mine from 20GB -> 50GB), let's
boot the VM and log into our guest. We'll use the _df_ command to examine our
mounted file filesystems:

    [root@fedora ~]# df -h
    Filesystem            Size  Used Avail Use% Mounted on
    /dev/mapper/VolGroup00-LogVol00
			   18G   15G  1.9G  89% /
    /dev/sda1             190M  163M   18M  91% /boot
    tmpfs                 502M   80K  502M   1% /dev/shm

Still shows 20 GB of total space and a filesystem device _/dev/mapper/VolGroup00-LogVol00_
that's indicative of LVM, the default for Fedora. Use the venerable _fdisk_ to examine
the "physical" disk:

    [root@fedora ~]# fdisk -l
    Disk /dev/sda: 52.6 GB, 52613349376 bytes
    255 heads, 63 sectors/track, 6396 cylinders
    Units = cylinders of 16065 * 512 = 8225280 bytes
    Disk identifier: 0x000f2b12
    Device    Boot      Start         End      Blocks   Id  System
    /dev/sda1   *           1          25      200781   83  Linux
    /dev/sda2              26        2610    20764012+  8e  Linux LVM

The physical disk has the capacity, but its in the form of unpartitioned/unformatted space.
Before we can do anything, We need to create a new partition containing this space.
First we'll drop into _parted_ and examine the existing partition table:

    (parted) print
    Model: VMware, VMware Virtual S (scsi)
    Disk /dev/sda: 52.6GB
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    Number  Start   End     Size    Type     File system  Flags
     1      32.3kB  206MB   206MB   primary  ext3         boot 
     2      206MB   21.5GB  21.3GB  primary               lvm  

Create a new partition that uses up all of the new space on the physical device:

    (parted) mkpart primary 21.4GB -1s 
    (parted) print
    Model: VMware, VMware Virtual S (scsi)
    Disk /dev/sda: 52.6GB
    Sector size (logical/physical): 512B/512B
    Partition Table: msdos
    Number  Start   End     Size    Type     File system  Flags
     1      32.3kB  206MB   206MB   primary  ext3         boot 
     2      206MB   21.5GB  21.3GB  primary               lvm  
     3      21.5GB  52.6GB  31.1GB  primary                    

Use _partprobe_ to load the table:

    (parted) quit
    [root@fedora ~]# partprobe

At this point we've got a new partition, _/dev/sda3_, that you could simply
format with _mke2fs_ and mount as its own filesystem e.g. _/mnt/data_. I
personally chose to leverage LVM's capabilities and incorporate the new
partition into the existing logical volume mounted as /. Start by examining
the set of existing physical volumes:

    [root@fedora ~]# lvm pvs
      PV         VG         Fmt  Attr PSize  PFree 
      /dev/sda2  VolGroup00 lvm2 a-   19.78G 32.00M

Create a new physical volume with the new partition:

    [root@fedora ~]# lvm pvcreate /dev/sda3
      Physical volume "/dev/sda3" successfully created
    [root@fedora ~]# lvm pvs
      PV         VG         Fmt  Attr PSize  PFree 
      /dev/sda2  VolGroup00 lvm2 a-   19.78G 32.00M
      /dev/sda3             lvm2 --   29.01G 29.01G

Add new physical volume to the volume group. If you recall from the initial
_df_ output above, the volume group in question is VolGroup0:

    [root@fedora ~]# lvm vgextend VolGroup00 /dev/sda3
      Volume group "VolGroup00" successfully extended
    [root@fedora ~]# lvm pvs
      PV         VG         Fmt  Attr PSize  PFree 
      /dev/sda2  VolGroup00 lvm2 a-   19.78G 32.00M
      /dev/sda3  VolGroup00 lvm2 a-   29.00G 29.00G

Now extend the logical volume to include the physical volume we just added to
the group. Use _lvm vgdisplay_ to examine the group:

    [root@fedora ~]# lvm vgdisplay VolGroup00
      --- Volume group ---
      VG Name               VolGroup00
      System ID
      Format                lvm2
      Metadata Areas        2
      Metadata Sequence No  4
      VG Access             read/write
      VG Status             resizable
      MAX LV                0
      Cur LV                2
      Open LV               2
      Max PV                0
      Cur PV                2
      Act PV                2
      VG Size               48.78 GB
      PE Size               32.00 MB
      Total PE              1561
      Alloc PE / Size       632 / 19.75 GB
      Free  PE / Size       929 / 29.03 GB
      VG UUID               8w2Wi9-T2lV-IKCV-fLRP-yTJl-teJM-LuGoNb

The _Free PE/Size_ field shows the new physical volume as 929 free extents.
Extend the logical volume to include all free extents:

    [root@fedora ~]# lvm lvextend -l+929 /dev/VolGroup00/LogVol00
      Extending logical volume LogVol00 to 46.81 GB
      Logical volume LogVol00 successfully resized

Use _resize2fs_ to extend our ext3 filesystem online. Note that many LVM
tutorials online refer to the now deprecated _ext2online_ command for this
step, _resize2fs_ now provides this functionality:

    root@fedora ~]# resize2fs /dev/VolGroup00/LogVol00 
    resize2fs 1.41.4 (27-Jan-2009)
    Filesystem at /dev/VolGroup00/LogVol00 is mounted on /; on-line resizing required
    old desc_blocks = 2, new_desc_blocks = 3
    Performing an on-line resize of /dev/VolGroup00/LogVol00 to 12271616 (4k) blocks.
    The filesystem on /dev/VolGroup00/LogVol00 is now 12271616 blocks long.

Finally, use _df_ again to verify that we have lots of free space:

    [root@fedora ~]# df -h
    Filesystem            Size  Used Avail Use% Mounted on
    /dev/mapper/VolGroup00-LogVol00
			   47G   15G   29G  34% /
    /dev/sda1             190M  163M   18M  91% /boot
    tmpfs                 502M   80K  502M   1% /dev/shm
