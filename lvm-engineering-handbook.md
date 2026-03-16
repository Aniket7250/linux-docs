LVM Engineering Handbook
Project: Atom-Bomb Storage Architecture

Author: Aniket Kumar
Purpose: Production-grade reference for creating, scaling, and troubleshooting Linux Logical Volume Manager (LVM) storage.

1. What is LVM

Logical Volume Manager (LVM) is a storage virtualization layer in Linux that allows flexible disk management.

Instead of binding filesystems directly to disks, LVM introduces a logical abstraction layer that enables:

Dynamic storage expansion

Disk pooling

Snapshot backups

Multi-disk aggregation

2. Storage Hierarchy

LVM works as a layered storage stack.

FILES
 │
 ▼
Filesystem (ext4 / xfs)
 │
 ▼
Logical Volume (LV)
 │
 ▼
Volume Group (VG)
 │
 ▼
Physical Volume (PV)
 │
 ▼
Physical Disk (/dev/sdb)
3. Operational Flow

Typical workflow when adding new storage.

Add Disk
   │
   ▼
Identify Disk
sudo lsblk
   │
   ▼
Create Physical Volume
sudo pvcreate /dev/sdb
   │
   ▼
Create Volume Group
sudo vgcreate atom-bomb /dev/sdb
   │
   ▼
Create Logical Volume
sudo lvcreate -L 5G -n molecules atom-bomb
   │
   ▼
Create Filesystem
sudo mkfs.ext4 /dev/atom-bomb/molecules
   │
   ▼
Mount Filesystem
sudo mount /dev/atom-bomb/molecules /mnt/physics_lab
   │
   ▼
Persist Mount
edit /etc/fstab
4. Storage Container Model (Venn Concept)
+----------------------------------------------------+
|                    Volume Group (VG)               |
|                                                    |
|   +--------------------------------------------+   |
|   |              Logical Volume (LV)           |   |
|   |                                            |   |
|   |   +------------------------------------+   |   |
|   |   |            Filesystem              |   |   |
|   |   |            (ext4/xfs)              |   |   |
|   |   |                                    |   |   |
|   |   |               FILES                |   |   |
|   |   |                                    |   |   |
|   |   +------------------------------------+   |   |
|   |                                            |   |
|   +--------------------------------------------+   |
|                                                    |
|  PV1 (/dev/sdb)      PV2 (/dev/sdc)                |
|                                                    |
+----------------------------------------------------+

Think of it like:

Disk → Storage Pool → Virtual Partition → Filesystem → Files
5. Step-by-Step Implementation
Identify Disks
sudo lsblk

Example output:

NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
sda       8:0    0  20G  0 disk
├─sda1    8:1    0 19.9G 0 part /
sdb       8:16   0  10G  0 disk

Interpretation

Device	Meaning
sda	OS disk
sda1	root filesystem
sdb	new disk for LVM
Create Physical Volume
sudo pvcreate /dev/sdb

Output

Physical volume "/dev/sdb" successfully created.

Verify

sudo pvs

Example

PV         VG   Fmt  Attr PSize   PFree
/dev/sdb        lvm2 ---  10.00g  10.00g
Create Volume Group
sudo vgcreate atom-bomb /dev/sdb

Output

Volume group "atom-bomb" successfully created

Verify

sudo vgs

Example

VG        #PV #LV #SN Attr   VSize  VFree
atom-bomb   1   0   0 wz--n- 10.00g 10.00g
Create Logical Volume
sudo lvcreate -L 5G -n molecules atom-bomb

Verify

sudo lvs

Example

LV        VG        Attr       LSize
molecules atom-bomb -wi-a----- 5.00g

Device path

/dev/atom-bomb/molecules

or

/dev/mapper/atom--bomb-molecules
Create Filesystem
sudo mkfs.ext4 /dev/atom-bomb/molecules

Example output

Creating filesystem with 1310720 4k blocks
Filesystem UUID: 7fad0a5a-f28b-4583-8f99-d9e9d1f43ab1
Mount Filesystem
sudo mkdir /mnt/physics_lab
sudo mount /dev/atom-bomb/molecules /mnt/physics_lab

Verify

df -h

Example

Filesystem                       Size Used Avail Use% Mounted on
/dev/atom-bomb/molecules         4.9G   24K  4.6G   1% /mnt/physics_lab
6. Persistent Mount

Find UUID

sudo blkid /dev/atom-bomb/molecules

Example

/dev/atom-bomb/molecules: UUID="7fad0a5a-f28b-4583-8f99-d9e9d1f43ab1"

Edit fstab

sudo nano /etc/fstab

Add

UUID=7fad0a5a-f28b-4583-8f99-d9e9d1f43ab1  /mnt/physics_lab  ext4  defaults  0 2

Validate

sudo mount -a
7. Expanding Storage

Extend LV

sudo lvextend -l +100%FREE /dev/atom-bomb/molecules

Resize filesystem

sudo resize2fs /dev/atom-bomb/molecules

Verify

df -h
8. Adding Additional Disks

Create new PV

sudo pvcreate /dev/sdc

Extend VG

sudo vgextend atom-bomb /dev/sdc

Now VG capacity increases.

9. LVM Snapshots

Create snapshot

sudo lvcreate -L 1G -s -n molecules-snap /dev/atom-bomb/molecules

Mount snapshot

sudo mount /dev/atom-bomb/molecules-snap /mnt/snapshot

Snapshots use copy-on-write.

10. LVM Internals
Extents

LVM divides disks into chunks called extents.

Default size

4MB
Physical Extent (PE)

Storage chunk inside a physical volume.

Logical Extent (LE)

Storage chunk assigned to logical volumes.

Mapping example

LV molecules

LE0 → /dev/sdb PE0
LE1 → /dev/sdb PE1
LE2 → /dev/sdc PE50
LE3 → /dev/sdc PE51

This allows LVM to span multiple disks.

11. LVM Metadata

Metadata exists in two places.

On Disk

Stored inside the PV header.

Inspect with

sudo pvdisplay -m

Backup metadata

sudo vgcfgbackup
In Kernel (Device Mapper)

LVM uses device mapper to create virtual block devices.

Example device

/dev/mapper/atom--bomb-molecules

Inspect

sudo dmsetup ls
12. IO Path

When a file is written:

Application
   ↓
Filesystem
   ↓
Logical Volume
   ↓
Device Mapper
   ↓
Physical Disk

Kernel path

write()
 ↓
VFS
 ↓
ext4
 ↓
block layer
 ↓
device mapper
 ↓
disk
13. Common Observations
Double Dash in Device Names
atom-bomb → atom--bomb

Because - is escaped internally.

Why 5GB appears as 4.9GB

Reasons

filesystem metadata

journaling

binary vs decimal units

This is expected.

14. Troubleshooting Commands

Check PV

sudo pvs

Check VG

sudo vgs

Check LV

sudo lvs

Detailed inspection

sudo pvdisplay
sudo vgdisplay
sudo lvdisplay
15. Recovery Scenario

If LVM volumes disappear:

sudo vgscan
sudo vgchange -ay

Check volumes

sudo lvscan
16. Engineer Cheat Sheet
sudo lsblk
sudo pvcreate /dev/sdb
sudo vgcreate atom-bomb /dev/sdb
sudo lvcreate -L 5G -n molecules atom-bomb
sudo mkfs.ext4 /dev/atom-bomb/molecules
sudo mkdir /mnt/physics_lab
sudo mount /dev/atom-bomb/molecules /mnt/physics_lab
sudo blkid
sudo mount -a
