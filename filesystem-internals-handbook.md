# Filesystem Internals Handbook
**Module:** 2 — Filesystem Internals
**Learner:** Aniket Kumar
**PERRIO Score:** 90/100
**Status:** Production-ready ✅

---

## Table of Contents
1. [The Big Picture](#1-the-big-picture)
2. [Inodes](#2-inodes)
3. [Block Pointers vs Extents](#3-block-pointers-vs-extents)
4. [Directories as Files](#4-directories-as-files)
5. [VFS — The Abstraction Layer](#5-vfs--the-abstraction-layer)
6. [Dentry Cache](#6-dentry-cache)
7. [Hard Links vs Soft Links](#7-hard-links-vs-soft-links)
8. [df vs du Divergence](#8-df-vs-du-divergence)
9. [Inode Exhaustion](#9-inode-exhaustion)
10. [ext4 vs XFS Decision Framework](#10-ext4-vs-xfs-decision-framework)
11. [fsck and Filesystem Repair](#11-fsck-and-filesystem-repair)
12. [Production Runbooks](#12-production-runbooks)
13. [Engineer Cheat Sheet](#13-engineer-cheat-sheet)

---

## 1. The Big Picture

### The Library Analogy
> - **Shelves** = data blocks (actual file content)
> - **Catalogue card** = inode (metadata + pointers to shelves)
> - **Card catalogue index** = directory entry (filename → inode number)
> - **Directory** = special file whose data blocks contain the name→inode table
> - **Librarian (kernel/OS)** = manages the catalogue, enforces access

### Storage Hierarchy
```
Application
    ↓  open() / write() / read()
   VFS  ← single interface for all filesystems
   ↙  ↓  ↘
ext4  xfs  tmpfs  nfs  btrfs
    ↓
Superblock → Inode → Data blocks
                ↑
         Directory (special file whose blocks = name→inode table)
```

### Two Independent Resources
```
A filesystem manages TWO separate pools:
1. Data blocks  → where file content lives
2. Inodes       → where file metadata lives

Both can run out independently.
Disk can have free blocks but zero free inodes → can't create new files.
Always check: df -h (blocks) AND df -i (inodes)
```

---

## 2. Inodes

### What an Inode Contains
```
✅ CONTAINS:                        ❌ DOES NOT CONTAIN:
   File size                           Filename (lives in directory)
   Owner, group                        Full path
   Permissions (rwx)                   File content
   Timestamps (atime/mtime/ctime)
   Link count
   Block pointers / extents
   File type (regular, dir, symlink)
```

### Link Count — How Deletion Works
```
Two conditions BOTH required to free data blocks:
1. Link count = 0  (no directory entry points to inode)
2. Open fd count = 0  (no process holds the file open)

create file.txt        → inode #48271, link count = 1
ln file.txt file2.txt  → inode #48271, link count = 2
rm file.txt            → link count = 1 (inode survives)
rm file2.txt           → link count = 0
                          BUT if nginx has it open → blocks still allocated
systemctl restart nginx → fd closed → kernel frees blocks → df updates
```

### Stat a File
```bash
stat filename          # full inode info
ls -i filename         # show inode number only
ls -i /var/app/cache/  # show inode numbers for directory contents
```

---

## 3. Block Pointers vs Extents

### Old Method (ext2/ext3) — Block Pointers
```
Inode (fixed 256 bytes):
├── 12 direct pointers      → point directly to data blocks
├── 1 single indirect       → block full of pointers
├── 1 double indirect       → block of blocks of pointers
└── 1 triple indirect       → one level deeper

Problem: pointer-chasing for large files
1GB file = 256,000 individual block pointer lookups
```

### Modern Method (ext4/XFS) — Extents
```
Extent = (start_block, length)

1GB sequential file:
  Old: 256,000 pointer entries
  New: 1 extent entry → "blocks 5000–261000"

Benefits:
✅ Less metadata overhead
✅ Faster sequential reads (one lookup, not 256,000)
✅ No pointer tree to traverse
✅ Single disk read for contiguous files
```

### Interview One-liner
> "Extents replace per-block pointer trees with start+length ranges — reducing metadata overhead and enabling large sequential reads without pointer-chasing."

---

## 4. Directories as Files

A directory is just a file whose data blocks contain a name→inode table:

```
Directory data block (on disk):
┌──────────┬──────────┬──────────┬────────────┐
│ inode_no │ rec_len  │ name_len │ filename   │
├──────────┼──────────┼──────────┼────────────┤
│ 48271    │ 16       │ 10       │ report.txt │
│ 48350    │ 12       │ 4        │ logs       │
│ 48102    │ 8        │ 6        │ .bashrc    │
└──────────┴──────────┴──────────┴────────────┘

rec_len = total record length (for jumping to next entry)
name_len = length of filename string
```

### ext4 HTree Index
- Large directories use HTree (hash tree) indexing
- Lookup is O(log n) not O(n)
- `ls` on 1M-file dir is slow, `stat specificfile` is fast

### mv vs cp vs hard link — What Changes
| Operation | Inode | Data blocks | Link count | What changes |
|---|---|---|---|---|
| `mv` (same FS) | Same | Same | Same | Directory entry only |
| `mv` (cross FS) | New | Copied | New=1 | Everything |
| `cp` | New | New | New=1 | New inode + new blocks |
| `ln` (hard) | Same | Same | +1 | New directory entry |
| `ln -s` (soft) | New | New (path string) | Unchanged | New inode with path |

> ⚠️ `mv` same FS ≠ hard link. Link count stays 1. Only the directory entry is rewritten. That's why rename is atomic and instant even for 100GB files.

---

## 5. VFS — The Abstraction Layer

```
Application calls write() → has no idea if it's ext4, XFS, tmpfs, or NFS

VFS defines four common objects every filesystem must implement:

┌─────────────┬──────────────────┬───────────────────────────────────────┐
│ Object      │ Lives in         │ Represents                            │
├─────────────┼──────────────────┼───────────────────────────────────────┤
│ superblock  │ Disk + memory    │ Mounted FS metadata (block size,      │
│             │                  │ inode count, free space)              │
├─────────────┼──────────────────┼───────────────────────────────────────┤
│ inode       │ Disk + memory    │ File/dir metadata + block pointers    │
├─────────────┼──────────────────┼───────────────────────────────────────┤
│ dentry      │ Memory only      │ Name→inode cache (path resolution)    │
├─────────────┼──────────────────┼───────────────────────────────────────┤
│ file        │ Memory only      │ One open instance per process per     │
│             │                  │ open() call                           │
└─────────────┴──────────────────┴───────────────────────────────────────┘
```

> 🎯 "How can Linux support so many filesystems?" → VFS defines the interface. Each filesystem implements it. The kernel only talks to VFS.

---

## 6. Dentry Cache

### What Dentry Is
```
Source of truth:  directory data blocks on DISK (permanent)
Dentry:           kernel's in-memory CACHE of name→inode mappings
                  (temporary, lost on reboot, rebuilt on access)
```

### Path Resolution with Dentry
```
Access: /home/aniket/report.txt

1. Look up dentry for "/"          → inode for root
2. Look up dentry for "home"       → inode #1001
3. Look up dentry for "aniket"     → inode #2045
4. Look up dentry for "report.txt" → inode #48271

Each step: check dcache first → cache hit = zero disk read
           cache miss = read directory data block → cache result
```

### Dentry States
```
Used     → currently referenced by a process
Unused   → no process using, kept in LRU cache
Negative → caches "this file does NOT exist"
           prevents repeated disk reads for missing files
```

### LRU Eviction
```
LRU = Least Recently Used
When RAM pressure hits → evict dentries accessed longest ago
Bet: recently used paths more likely to be needed again
```

> ⚠️ Heavy stat() calls on non-existent paths → negative dentry accumulation → memory pressure. Real production performance issue.

---

## 7. Hard Links vs Soft Links

```
Hard link:                           Soft link (symlink):
──────────                           ────────────────────
name_a ──┐                           link ──► inode ──► data block
          ├──► inode ──► data blocks              containing PATH STRING
name_b ──┘                                        e.g. "/home/aniket/real.txt"
                                                  resolved at access time
Same inode number                    Different inode number
Link count increments (+1)           Target link count unchanged
Cannot cross filesystems             Can cross filesystems
Cannot link directories              Can link directories
Target deleted = data survives       Target deleted = dangling symlink
                                     "No such file or directory"
```

> ⚠️ Soft link stores a PATH STRING in its data block — NOT a pointer to the target's data blocks. This is the most common misconception.

---

## 8. df vs du Divergence

```bash
# Always check both
df -h          # block space
df -i          # inode usage ← never forget this one

# df shows MORE than du:
lsof | grep deleted                    # deleted files held open by processes
lsof | grep deleted | awk '{print $1, $2, $7}'  # process, PID, size
tune2fs -l /dev/sda1 | grep Reserved  # ext4 5% reserved for root
                                       # fix: tune2fs -m 1 /dev/vg/lv
find /dir -type f -sparse              # sparse files

# Fix deleted-but-open:
systemctl restart <service>            # closes old fd → kernel frees blocks
kill -HUP <pid>                        # if service supports SIGHUP log reopen
```

### The Deleted-but-open Pattern
```
rm nginx.log          → link count 0, nginx still has fd open
                        inode alive, blocks allocated, df shows used
systemctl restart nginx → nginx closes old fd
                        → kernel frees blocks → df updates

Diagnose: lsof | grep deleted
```

---

## 9. Inode Exhaustion

### Detection
```bash
df -i                              # check IUse% on all filesystems
df -i | awk '$5 == "100%"'        # find exhausted filesystems
find / -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head -10
# ↑ find directory with most files
```

### Prevention at mkfs Time (ext4)
```bash
# Default: 1 inode per 16KB → use for large files
mkfs.ext4 /dev/sdb

# Tiny files (mail, cache, sessions): 1 inode per 4KB
mkfs.ext4 -i 4096 /dev/sdb

# Large files (video, backups): 1 inode per 512KB
mkfs.ext4 -i 524288 /dev/sdb

# Block size tuning (separate axis):
mkfs.ext4 -b 1024 /dev/sdb       # 1KB blocks — less waste per tiny file
                                   # tradeoff: more metadata overhead
mkfs.ext4 -b 4096 /dev/sdb       # 4KB default — good for general use
```

### ext4 vs XFS on Inodes
```
ext4: inode count FIXED at mkfs — cannot change without reformat
      over-provision for tiny-file workloads or reformat later

XFS:  inodes allocated DYNAMICALLY from data pool
      no upfront guessing required
      rarely exhausted — recommended for unpredictable workloads
```

### Performance Impact of High Inode Count
```
More inodes → larger inode table → slower fsck
           → more inode cache memory pressure
           → larger directory → slower lookup
           → wasted disk space for unused inode slots
```

### Monitoring
```bash
# Alert at 70% — not 100%
df -i | awk 'NR>1 && $5+0 > 70 {print $6, $5}'

# Add to Prometheus node_exporter custom collector or nagios check
```

---

## 10. ext4 vs XFS Decision Framework

| Workload | Filesystem | Key Tuning |
|---|---|---|
| Millions of tiny files | **XFS** | dynamic inodes — no tuning needed |
| Tiny files, ext4 required | ext4 | `-i 4096 -b 1024` |
| Large sequential files (video, DB) | **XFS** | default + RAID stripe alignment |
| General purpose | Either | XFS default on RHEL 9 |
| Boot/root partition | ext4 | wider tooling support |
| Cloud single disk | Either | cloud-native resize, skip LVM |
| Bare metal multi-disk | **XFS + LVM + RAID** | RAID for redundancy, LVM for flex |

### RAID + LVM + XFS Stack (Bare Metal Production)
```
Layer 1: RAID 10 (mdadm)          → redundancy + read performance
Layer 2: LVM on top of RAID       → flexibility, snapshots, online resize
Layer 3: XFS                      → large file performance, dynamic inodes

mkfs.xfs -d su=64k,sw=4 /dev/md0  # align XFS stripe to RAID geometry
                                    # eliminates write amplification
                                    # su = stripe unit, sw = stripe width
```

---

## 11. fsck and Filesystem Repair

### What fsck Checks
```
Superblock    → is FS metadata valid?
Inode table   → are all inodes consistent?
Block bitmap  → do allocated blocks match inodes?
Directory tree → are all entries valid?
Link counts   → do they match actual directory entries?
```

### Commands
```bash
# ext4
fsck /dev/sda1              # check and repair (unmounted only)
fsck -n /dev/sda1           # dry run — check only, no fixes

# XFS
xfs_repair /dev/sda1        # repair (unmounted only)
xfs_repair -n /dev/sda1     # dry run

# Inspect
tune2fs -l /dev/sda1        # ext4 superblock info
xfs_info /mnt/data          # XFS superblock info
dumpe2fs /dev/sda1          # detailed ext4 dump
```

> ⚠️ Never run fsck on a live mounted filesystem — data corruption risk.
> Journaling (ext4/XFS) makes most unclean shutdowns fast via journal replay.

---

## 12. Production Runbooks

### Runbook: Disk Full but du Shows Free Space
```bash
# 1. Check both resources
df -h && df -i

# 2. Find deleted-but-open files
lsof | grep deleted
lsof | grep deleted | awk '{print $1, $2, $7}'  # process, PID, size

# 3. Restart offending service
systemctl restart <service>

# 4. Verify recovery
df -h && df -i
```

### Runbook: Inode Exhaustion
```bash
# 1. Confirm
df -i | awk '$5 == "100%"'

# 2. Find culprit directory
find / -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head -10

# 3. Quick relief — delete expired files
find /var/app/cache -name "*.session" -mtime +1 | wc -l  # confirm count first
find /var/app/cache -name "*.session" -mtime +1 -delete

# 4. Check for deleted-but-open inodes
lsof | grep deleted | grep session

# 5. Watch recovery
watch -n 2 'df -i /var/app/cache'

# 6. Long term — reformat with better ratio
# Backup → lvremove → lvcreate → mkfs.ext4 -i 4096 or mkfs.xfs → restore
```

### Runbook: Can't Create Files Despite Free Space
```bash
# Always check inodes first
df -i                          # if IUse% = 100% → inode exhaustion
df -h                          # if Use% = 100% → block exhaustion

# Inode exhaustion path → see above runbook
# Block exhaustion path → see LVM expansion runbook (Module 1)
```

---

## 13. Engineer Cheat Sheet

```bash
# INSPECT
stat filename                  # full inode info
ls -i filename                 # inode number
df -h && df -i                 # blocks AND inodes (always both)
tune2fs -l /dev/sda1           # ext4 superblock
xfs_info /mnt/data             # XFS info
lsof | grep deleted            # deleted-but-open files

# INODE TUNING (at mkfs time)
mkfs.ext4 -i 4096 /dev/sdb    # more inodes (tiny files)
mkfs.ext4 -i 524288 /dev/sdb  # fewer inodes (large files)
mkfs.ext4 -b 1024 /dev/sdb    # smaller blocks (less waste per tiny file)
mkfs.xfs /dev/sdb              # dynamic inodes — no tuning needed
mkfs.xfs -d su=64k,sw=4 /dev/md0  # RAID stripe alignment

# LINKS
ln file.txt hardlink.txt       # hard link — same inode, link count +1
ln -s /path/to/file symlink    # soft link — new inode, stores path string
readlink symlink               # show what symlink points to

# REPAIR
fsck -n /dev/sda1              # ext4 dry run check
xfs_repair -n /dev/sda1        # XFS dry run check

# FIND FILES
find /dir -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head
find /dir -name "*.session" -mtime +1 | wc -l
find /dir -name "*.session" -mtime +1 -delete
```

### Critical Rules
| Rule | Detail |
|---|---|
| Filename NOT in inode | Lives in directory data block |
| `mv` same FS | Only dentry changes — link count stays 1 |
| Soft link stores PATH | Not pointer to data blocks — resolves at access time |
| Two conditions to free blocks | Link count = 0 AND open fd count = 0 |
| Always check df -i | Inode exhaustion looks identical to block exhaustion |
| ext4 inodes fixed at mkfs | Cannot change without reformat |
| XFS inodes dynamic | Allocates from data pool on demand |
| Block size tradeoff | Smaller blocks = less waste, more metadata overhead |
| dentry is cache only | Lost on reboot, source of truth is disk |
| fsck needs unmounted FS | Live fsck = data corruption |

---

*Handbook built during PERRIO Module 2 session*
*Overall score: 90/100 — Production-ready on Filesystem Internals*
*Next module: RAID (Module 3)*
