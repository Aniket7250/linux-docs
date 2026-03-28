
# PERRIO Learning Context — LVM
**Topic:** Linux Logical Volume Manager (LVM)
**Source Handbook:** lvm-engineering-handbook.md (Atom-Bomb Storage Architecture)
**Learner:** Aniket Kumar
**Goal:** 70–80 LPA SRE Role

---

## Session Progress

| Stage | Status |
|-------|--------|
| 1 — Priming | ✅ Complete |
| 2 — Encoding | ✅ Complete |
| 3 — Reference | ✅ Complete |
| 4 — Retrieval | ✅ Complete |
| 5 — Interleaving | ✅ Complete |
| 6 — Overlearning | ✅ Complete |

---

## Stage 1 — Priming Notes

### Analogy Used
- Raw disks = awkwardly shaped storage rooms
- LVM = storage architect who knocks walls, creates open-plan space with pencil partitions

### Core Mental Model (Aniket's Own Words)
```
PD1 + PD2 + PD3
↓
PV1(PD1) + PV2(PD2) + PV3(PD3)
↓
VG[ PV1 + PV2 + PV3 ]
↓
LV { VG } + LV1 { VG }
↓
FS on each LV
```

### Curiosity Questions — Aniket's Answers
1. **Can you span 20GB across two 10GB disks?**
   - ✅ Correct: PV → VG → LV → FS. Noted filesystem overhead means < 20GB usable.

2. **What happens if one disk fails mid-span?**
   - ⚠️ Partially correct. Aniket thought data might survive on other PDs.
   - **Correction:** LVM default = Linear mode. If PV1 fails, all blocks on it are LOST. No automatic redundancy. Need explicit LVM mirroring or RAID.

3. **Why do cloud VMs use LVM on root?**
   - ✅ Correct: Online scalability, cost management, fault tolerance, live resize without reboot.

### Corrections Given
- ❌ LVM Linear ≠ Fault Tolerant (no automatic redundancy)
- ❌ FS wraps LV (wrong) → FS is formatted ONTO LV like a house built ON a plot

### Corrected Mental Model
```
Physical Disks:    [PD1]   [PD2]   [PD3]
                     ↓       ↓       ↓
Physical Volumes:  [PV1]  [PV2]  [PV3]
                     └───────┴───────┘
Volume Group:            [VG]
                        ↙    ↘
Logical Volumes:      [LV1]  [LV2]
                        ↓      ↓
Filesystems:         [ext4] [xfs]
                        ↓      ↓
Mount Points:      /mnt/data  /mnt/logs
```

---

## Key Concepts to Remember

- **LVM default = Linear** (no redundancy). Redundancy requires mirroring or RAID.
- **VG = pool** of all PV capacity combined
- **LV = virtual partition** carved from VG — flexible, resizable
- **FS sits ON TOP of LV** — LV is just a block device
- **XFS cannot be shrunk** — only grown (critical production gotcha)

---

## Stage 2 — Encoding

### Aniket's Score: 95/100

### Key Concepts Learned

**Extents**
- PE (Physical Extent) = 4MB tile on a physical disk
- LE (Logical Extent) = same tile, named from the LV's perspective
- LV size = requested size ÷ 4MB = number of extents claimed from VG pool
- LE and PE map 1:1 in linear mode

**Mapping Table**
- LVM maintains an internal table: LE → which disk → which PE
- Allows LV to span multiple disks while appearing contiguous to filesystem
- ❌ Interview trap: "Is an LV contiguous on disk?" → NO. Logically yes, physically fragmented.

**Device Mapper (DM)**
- Kernel subsystem that creates virtual block devices
- LVM programs DM with the mapping table
- DM translates: LE (logical) → PE (physical) → routes IO to correct disk
- Direction: always Logical → Physical (not PE→LE)
- `atom-bomb` becomes `atom--bomb` in DM (dash escaping to avoid ambiguity)
- Commands: `sudo dmsetup ls` / `sudo dmsetup table`

**XFS Cannot Shrink — Why**
- XFS writes Allocation Group (AG) metadata headers at FIXED offsets inside the volume
- Shrinking the block device destroys those headers → filesystem corrupted
- ext4 metadata is relocatable → can shrink offline
- ✅ SRE Rule: Grow XFS in small increments. You cannot undo it.
- ✅ Recovery if over-allocated: Backup → unmount → `lvremove` → `lvcreate` (smaller) → `mkfs.xfs` → restore

**IO Path (write)**
```
write() → VFS → XFS/ext4 (file→block addresses) → Block Layer (queue/schedule)
→ Device Mapper (LE→PE translation) → Disk Driver → Physical Sectors
```

**Snapshots (Copy-on-Write)**
- Snapshot = pointer to original LV state, not a full copy
- On modification: old block copied to snap store FIRST, then new block written to original
- Space efficient: only changed blocks stored
- Use case: pre-migration safety net, rollback in seconds vs hours

### Corrections Given in Stage 2
- ⚠️ IO path direction: DM translates LE→PE (not PE→LE)
- ⚠️ XFS shrink recovery: destroy the LV (not "remove the disk") — `lvremove` then recreate

---

## Stage 3 — Reference Cheat Sheet

### LVM Hierarchy
```
PD → PV (pvcreate) → VG (vgcreate) → LV (lvcreate) → FS (mkfs) → Mount
```

### Commands
```bash
# CREATE
sudo pvcreate /dev/sdb
sudo vgcreate atom-bomb /dev/sdb
sudo lvcreate -L 5G -n molecules atom-bomb
sudo mkfs.xfs /dev/atom-bomb/molecules
sudo mkdir -p /mnt/physics_lab
sudo mount /dev/atom-bomb/molecules /mnt/physics_lab

# INSPECT
lsblk -f
sudo pvs / pvdisplay
sudo vgs / vgdisplay
sudo lvs / lvdisplay
sudo dmsetup ls
sudo dmsetup table

# EXPAND
sudo pvcreate /dev/sdc
sudo vgextend atom-bomb /dev/sdc
sudo lvextend -r -L +5G /dev/atom-bomb/molecules     # -r = auto resize FS

# Manual XFS expand:
sudo lvextend -L +5G /dev/atom-bomb/molecules
sudo xfs_growfs /mnt/physics_lab     # mount point, NOT device path

# Manual ext4 expand:
sudo lvextend -L +5G /dev/atom-bomb/molecules
sudo resize2fs /dev/atom-bomb/molecules   # device path, NOT mount point

# PERSIST (fstab)
sudo blkid /dev/atom-bomb/molecules
# UUID=xxxx  /mnt/physics_lab  xfs  defaults  0 2
sudo mount -a

# SNAPSHOTS
sudo lvcreate -L 1G -s -n molecules-snap /dev/atom-bomb/molecules
sudo mount /dev/atom-bomb/molecules-snap /mnt/snapshot

# RECOVERY
sudo vgscan && sudo vgchange -ay && sudo lvscan
```

### Critical Rules
| Rule | Detail |
|---|---|
| XFS cannot shrink | Backup → lvremove → lvcreate → mkfs → restore |
| xfs_growfs target | Mount point (/mnt/...) |
| resize2fs target | Device path (/dev/...) |
| LVM linear ≠ redundant | Disk failure = data loss on that PV |
| Default PE size | 4MB |
| Snapshot = CoW | Only changed blocks stored |
| DM double-dash | atom-bomb → atom--bomb |

### IO Path One-liner
```
write() → VFS → XFS/ext4 → Block Layer → Device Mapper (LE→PE) → Disk Driver → Sectors
```

---

## Stage 4 — Retrieval ✅ Complete

### Score: 88/100

| Q | Score | Notes |
|---|---|---|
| Q1 — lvextend across PVs | 90/100 | Correct. Addition: use `/dev/sdb` arg to force specific PV |
| Q2 — xfs_growfs wrong path | 100/100 | Perfect |
| Q3 — snapshot vs full backup | 90/100 | Addition: CoW store grows over time; if it fills up, snapshot becomes invalid |
| Q4 — 3AM disk full incident | 85/100 | Missing: `lvs`/`vgs` to get LV name first; use `-r` flag on lvextend |
| Q5 — XFS shrink procedure | 85/100 | Missing key word: `lvremove` to destroy LV, not the disk |

### Corrected 3AM Procedure (Q4)
```bash
sudo pvs && sudo lvs && sudo vgs             # 1. identify names
sudo pvcreate /dev/sdc                       # 2. init new disk
sudo vgextend <vgname> /dev/sdc              # 3. add to VG
sudo lvextend -r -L +XG /dev/vg/lv          # 4. extend LV + FS (-r handles xfs_growfs)
df -h /mnt/postgres-data                    # 5. verify
```

### Corrected XFS Shrink Procedure (Q5)
```bash
tar -czf /backup/data.tar.gz /mnt/physics_lab   # 1. backup
sudo umount /mnt/physics_lab                     # 2. unmount
sudo lvremove /dev/atom-bomb/molecules           # 3. destroy LV
sudo lvcreate -L 20G -n molecules atom-bomb      # 4. recreate smaller
sudo mkfs.xfs /dev/atom-bomb/molecules           # 5. reformat
sudo mount /dev/atom-bomb/molecules /mnt/physics_lab
tar -xzf /backup/data.tar.gz -C /mnt/physics_lab # 6. restore
```
## Stage 5 — Interleaving ✅ Complete

### Score: 100/100

**Scenario 1:** RAID for disk redundancy + LVM on top for snapshots = correct production pattern for bare metal DB
**Scenario 2:** Cloud console/CLI resize is simpler than LVM for single EBS volume — LVM only justified if pooling multiple volumes or need LVM-specific snapshots

### Key Comparisons Learned
- LVM vs RAID: use together — RAID for redundancy, LVM for flexibility
- LVM vs ZFS: ZFS has built-in snapshots/RAID/checksumming but higher complexity, not kernel-native
- When NOT to use LVM: single cloud managed disk — just use cloud-native resize
## Stage 6 — Overlearning ✅ Complete

### Score: 92/100

### Incident: /mnt/postgres-data at 97%, unused /dev/sdc available

### What Aniket did well
- Used `wipefs -a` before pvcreate — senior move, clears old signatures that cause pvcreate to fail
- Pulled correct VG name `datavg` from incident data without being told
- Ran `vgs` to validate VFree before extending
- Self-corrected `-r` flag mistake mid-answer under pressure
- Instantly diagnosed xfs_growfs device path error (Q3)

### Corrections
| Item | Issue | Fix |
|---|---|---|
| `lsblk /dev/sdc` | Missing `-f` flag | `lsblk -f /dev/sdc` — shows FS signatures, confirms disk is truly blank |
| `-L +100G` | Assumes exact free space | Use `-l +100%FREE` — claims all available extents safely |
| Mapper path typo | `/dev/mapper/datavg/-pgdata` | `/dev/mapper/datavg-pgdata` or tab-complete always |

### Correct Incident Runbook
```bash
sudo lsblk -f /dev/sdc                          # 1. confirm disk is clean (shows FS signatures)
sudo wipefs -a /dev/sdc                         # 2. wipe old signatures
sudo pvcreate /dev/sdc                          # 3. init PV
sudo vgextend datavg /dev/sdc                   # 4. add to VG
sudo vgs                                        # 5. verify VFree increased
sudo lvextend -l +100%FREE /dev/datavg/pgdata   # 6. claim all free extents
sudo xfs_growfs /mnt/postgres-data              # 7. grow XFS — mount point, never device path
df -h /mnt/postgres-data                        # 8. verify
```

### Key Production Rules
- `wipefs` before pvcreate on reused disks — always
- `-l +100%FREE` in incidents — don't guess size under pressure
- `xfs_growfs` = mount point | `resize2fs` = device path — never mix these
- Tab-complete mapper paths — never type `/dev/mapper/...` manually

---

## Final Scores

| Stage | Score |
|---|---|
| Priming | 85/100 |
| Encoding | 95/100 |
| Retrieval | 88/100 |
| Interleaving | 100/100 |
| Overlearning | 92/100 |
| **Overall** | **92/100** |

**Status: Production-ready on LVM ✅**