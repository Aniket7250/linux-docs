# Linux SRE Learning Roadmap
**Learner:** Aniket Kumar
**Goal:** 70–80 LPA SRE Role
**Method:** PERRIO Framework (Priming → Encoding → Reference → Retrieval → Interleaving → Overlearning)
**Last Updated:** Session 1

---

## How This Works

- Each module has a **status**, **score**, and **link to its handbook**
- After every PERRIO session, update the status and score here
- Each module gets its own handbook file (like `lvm-engineering-handbook.md`)
- The PERRIO context file tracks your answers, corrections, and session notes

---

## Progress Overview

| Tier | Module | Status | Score | Handbook |
|------|--------|--------|-------|----------|
| Foundation | Storage & LVM | ✅ Complete | 92/100 | [lvm-engineering-handbook.md](./lvm-engineering-handbook.md) |
| Foundation | Filesystem Internals | ✅ Complete | 90/100 | [filesystem-internals-handbook.md](./filesystem-internals-handbook.md) |
| Foundation | RAID | ⏳ Not started | — | — |
| OS Internals | systemd | ⏳ Not started | — | — |
| OS Internals | Processes & Signals | ⏳ Not started | — | — |
| OS Internals | Memory Management | ⏳ Not started | — | — |
| Networking | Network Fundamentals | ⏳ Not started | — | — |
| Networking | Firewall & Security | ⏳ Not started | — | — |
| Networking | DNS & Load Balancing | ⏳ Not started | — | — |
| Observability | Logging | ⏳ Not started | — | — |
| Observability | Performance Metrics | ⏳ Not started | — | — |
| Observability | Tracing & Profiling | ⏳ Not started | — | — |
| SRE Practice | Troubleshooting | ⏳ Not started | — | — |
| SRE Practice | Security Hardening | ⏳ Not started | — | — |
| SRE Practice | Automation & Scripting | ⏳ Not started | — | — |

---

## Tier 1 — Foundation

> Core storage and filesystem knowledge. Every SRE touches this daily.

### ✅ Module 1: Storage & LVM
**Status:** Complete | **Score:** 92/100
**Handbook:** [lvm-engineering-handbook.md](./lvm-engineering-handbook.md)
**PERRIO Context:** [PERRIO-Context.md](./PERRIO-Context.md)

**What you mastered:**
- PV → VG → LV → FS hierarchy and the extent model (PE/LE, 4MB default)
- Device Mapper as the kernel engine translating LE→PE
- XFS vs ext4: grow tools, shrink rules, and why XFS headers are fixed
- Full IO path: write() → VFS → FS → Block Layer → DM → Disk
- LVM snapshots via Copy-on-Write
- Production runbook: disk full incident with wipefs, pvcreate, vgextend, lvextend

**Key gotchas to remember:**
- `xfs_growfs` → mount point | `resize2fs` → device path
- LVM linear ≠ fault tolerant (no automatic redundancy)
- XFS cannot shrink — ever. Backup → lvremove → recreate → restore
- Use `-l +100%FREE` in incidents, not `-L +XG`
- Always `wipefs -a` before `pvcreate` on reused disks

---

### ✅ Module 2: Filesystem Internals
**Status:** Complete | **Score:** 90/100
**Handbook:** [filesystem-internals-handbook.md](./filesystem-internals-handbook.md)

**What you mastered:**
- Inode structure — what's in it, what's not (filename lives in directory)
- Block pointers vs extents — why extents win for large files
- VFS four objects: superblock, inode, dentry, file
- Dentry cache — LRU, negative dentries, path resolution mechanics
- df vs du divergence — deleted-but-open, reserved blocks, sparse files
- Inode exhaustion — detection, prevention, two tuning axes (-i ratio, -b block size)
- Hard link vs soft link — link count mechanics, path string vs data pointer
- ext4 vs XFS decision framework — when to use each

**Key gotchas to remember:**
- `mv` same FS = dentry rewrite only, link count stays 1 (NOT a hard link)
- Soft link stores PATH STRING — not a pointer to data blocks
- Always `df -h` AND `df -i` — inode exhaustion looks identical to block exhaustion
- ext4 inode count fixed at mkfs — XFS allocates dynamically
- Two conditions to free blocks: link count = 0 AND open fd count = 0

---

### ⏳ Module 3: RAID
**Status:** Not started
**Planned handbook:** `raid-handbook.md`

**What we'll cover:**
- RAID levels: 0, 1, 5, 6, 10 — math, tradeoffs, failure tolerance
- mdadm: create, assemble, monitor, repair arrays
- RAID vs LVM vs ZFS — when to use what
- Rebuild times, write hole problem, RAID 5 vs 6 for large disks
- Hardware RAID vs software RAID

**Why it matters for SRE:**
- Bare metal DB servers almost always run RAID + LVM
- Knowing rebuild impact on production IO
- Handling degraded array alerts at 3AM

---

## Tier 2 — OS Internals

> How Linux works under the hood. This is what separates SREs from sysadmins.

### ⏳ Module 4: systemd
**Status:** Not started
**Planned handbook:** `systemd-handbook.md`

**What we'll cover:**
- Unit types: service, socket, timer, mount, target
- Service lifecycle: start, stop, restart, reload, enable
- journald: structured logging, filtering, persistence
- Targets vs runlevels, boot sequence
- Writing clean service unit files
- systemd timers vs cron
- Dependency ordering: Wants, Requires, After, Before

**Why it matters for SRE:**
- Every service you manage runs under systemd
- Debugging failed units at boot
- Replacing cron with reliable systemd timers

---

### ⏳ Module 5: Processes & Signals
**Status:** Not started
**Planned handbook:** `processes-signals-handbook.md`

**What we'll cover:**
- fork(), exec(), wait() — process lifecycle
- Signals: SIGTERM, SIGKILL, SIGHUP, SIGCHLD — what each does
- Process states: R, S, D, Z — zombie and uninterruptible sleep
- ps, top, htop, pgrep, kill, strace
- File descriptors, /proc filesystem
- Namespaces and cgroups (bridge to containers)

**Why it matters for SRE:**
- "Process won't die" — knowing SIGTERM vs SIGKILL
- D-state processes blocking on IO — how to diagnose
- strace for black-box debugging

---

### ⏳ Module 6: Memory Management
**Status:** Not started
**Planned handbook:** `memory-management-handbook.md`

**What we'll cover:**
- Virtual memory, page tables, TLB
- RSS vs VSZ vs PSS — what each means
- Page cache, buffer cache — why free memory isn't wasted
- Swap: when it's used, swappiness, swap pressure
- OOM killer: how it picks victims, oom_score_adj
- cgroups memory limits and containment
- Memory leaks: detecting with /proc/meminfo, valgrind

**Why it matters for SRE:**
- OOM kills in production — root cause and prevention
- Tuning swappiness for database servers
- "System has 64GB RAM but app is slow" — page cache behavior

---

## Tier 3 — Networking

> The plumbing of distributed systems. Critical for any SRE role.

### ⏳ Module 7: Network Fundamentals
**Status:** Not started
**Planned handbook:** `networking-fundamentals-handbook.md`

**What we'll cover:**
- OSI model and where Linux tools map to each layer
- TCP/IP: handshake, teardown, TIME_WAIT, connection states
- ip, ss, netstat — reading connection tables
- Routing tables, default gateway, policy routing
- Network namespaces (bridge to Kubernetes networking)
- tcpdump and wireshark basics
- Bandwidth vs latency — tuning for each

**Why it matters for SRE:**
- "Service can't reach DB" — systematic diagnosis path
- TIME_WAIT accumulation killing connection pools
- Understanding what Kubernetes networking builds on

---

### ⏳ Module 8: Firewall & Security
**Status:** Not started
**Planned handbook:** `firewall-security-handbook.md`

**What we'll cover:**
- Netfilter and the kernel packet path
- iptables: tables, chains, rules, NAT
- nftables: modern replacement syntax
- firewalld: zones and services abstraction
- SELinux: enforcing vs permissive, contexts, audit2allow
- conntrack: stateful packet inspection

**Why it matters for SRE:**
- Debugging "connection refused vs connection timed out" — the firewall signal
- Kubernetes kube-proxy uses iptables/ipvs
- SELinux denials blocking legitimate traffic at 3AM

---

### ⏳ Module 9: DNS & Load Balancing
**Status:** Not started
**Planned handbook:** `dns-loadbalancing-handbook.md`

**What we'll cover:**
- DNS resolution chain: resolver → recursive → authoritative
- /etc/resolv.conf, nsswitch.conf, systemd-resolved
- Common DNS failure modes: NXDOMAIN, SERVFAIL, TTL traps
- HAProxy: frontend, backend, ACLs, health checks
- nginx as reverse proxy and load balancer
- Upstream health checks and circuit breaking

**Why it matters for SRE:**
- DNS is involved in almost every production incident
- "Works on my machine" often means DNS inconsistency
- Load balancer config is modified during every deployment

---

## Tier 4 — Observability

> You can't fix what you can't see. Observability is the SRE superpower.

### ⏳ Module 10: Logging
**Status:** Not started
**Planned handbook:** `logging-handbook.md`

**What we'll cover:**
- syslog protocol and facility/severity levels
- rsyslog: config, filters, remote shipping
- journald: binary format, persistence, export
- Log rotation: logrotate config, copytruncate vs create
- Structured logging: JSON logs and parsing with jq
- Log aggregation patterns (ship to ELK, Loki, etc.)

**Why it matters for SRE:**
- Logs are the first thing you open in any incident
- Missing logs = missing context = slower resolution
- Log volume and rotation misconfig causes disk full incidents

---

### ⏳ Module 11: Performance Metrics
**Status:** Not started
**Planned handbook:** `performance-metrics-handbook.md`

**What we'll cover:**
- The USE Method: Utilization, Saturation, Errors — per resource
- CPU: vmstat, mpstat, top, context switches, steal time
- Memory: /proc/meminfo, free, slab cache
- Disk IO: iostat, iotop, await, %util — what saturation looks like
- Network: sar -n, ss, dropped packets, retransmits
- eBPF basics: what it enables that classic tools can't

**Why it matters for SRE:**
- USE method is the systematic answer to "system is slow"
- CPU steal time on cloud VMs — noisy neighbor problem
- Disk await > 50ms = latency spike in any app

---

### ⏳ Module 12: Tracing & Profiling
**Status:** Not started
**Planned handbook:** `tracing-profiling-handbook.md`

**What we'll cover:**
- strace: syscall tracing, reading output, common patterns
- ltrace: library call tracing
- perf: CPU profiling, PMU counters, perf stat
- Flame graphs: reading, generating with perf + flamegraph.pl
- eBPF tracing: bpftrace one-liners, BCC tools
- Continuous profiling concepts (pprof, pyroscope)

**Why it matters for SRE:**
- Black-box debugging: app is slow but you have no source code
- Flame graphs are the fastest way to find CPU hotspots
- eBPF is replacing most tracing tools in modern SRE stacks

---

## Tier 5 — SRE Practice

> Where all the knowledge comes together under real pressure.

### ⏳ Module 13: Troubleshooting Methodology
**Status:** Not started
**Planned handbook:** `troubleshooting-handbook.md`

**What we'll cover:**
- The USE Method and RED Method — when to use each
- Systematic diagnosis: start broad, narrow fast
- Runbook writing: anatomy of a good runbook
- Incident response: roles, communication, timelines
- Post-mortems: blameless culture, 5 whys, action items
- Common production patterns: CPU bound, IO bound, memory pressure, network saturation

**Why it matters for SRE:**
- Methodology under pressure separates senior from junior
- Interviewers simulate incidents — this is your playbook
- Runbooks are the artifact that makes you promotable

---

### ⏳ Module 14: Security Hardening
**Status:** Not started
**Planned handbook:** `security-hardening-handbook.md`

**What we'll cover:**
- SSH hardening: key auth, AllowUsers, port, fail2ban
- sudo configuration: sudoers, NOPASSWD traps, least privilege
- PAM: pluggable auth modules, password policies
- auditd: syscall auditing, audit rules, ausearch
- CIS benchmarks: what they are and how to apply them
- Secrets management: vault patterns, env var anti-patterns

**Why it matters for SRE:**
- Security incidents are SRE incidents
- Hardening is part of every production onboarding checklist
- Audit logs are what you read after a breach

---

### ⏳ Module 15: Automation & Scripting
**Status:** Not started
**Planned handbook:** `automation-scripting-handbook.md`

**What we'll cover:**
- Bash scripting: conditionals, loops, functions, error handling
- set -euo pipefail — why every script needs this
- cron: syntax, pitfalls, debugging missed jobs
- systemd timers: calendar expressions, accuracy vs cron
- Ansible basics: inventory, playbooks, idempotency
- Scripting patterns for SRE: health checks, auto-remediation

**Why it matters for SRE:**
- Manual = not scalable. Automation = promotable
- Bad scripts in cron cause silent production failures
- Ansible is in almost every SRE job description

---

## Session Log

| Date | Module | Stage Reached | Score | Notes |
|------|--------|---------------|-------|-------|
| Session 1 | Storage & LVM | All 6 stages | 92/100 | Strong on XFS internals and production runbook. Watch: device path vs mount point under pressure |
| Session 2 | Filesystem Internals | All 6 stages | 90/100 | Strong on df/du divergence and inode exhaustion. Watch: mv ≠ hard link, soft link stores path string not data pointer |

---

## Key Cross-Module Connections

```
LVM ──────────────► Filesystem Internals
  (LV is a block device; FS formats it)

RAID ─────────────► LVM
  (RAID underneath for redundancy, LVM on top for flexibility)

Filesystem ──────► systemd
  (systemd mount units replace fstab entries)

Processes ───────► Memory Management
  (each process has virtual address space, page tables)

Memory ──────────► Performance Metrics
  (page cache, OOM, swappiness all show up in metrics)

Networking ──────► Troubleshooting
  (half of all incidents involve network diagnosis)

All tiers ───────► Troubleshooting Methodology
  (USE method applies to every resource type)
```

---

## Interview Priority Order

If you have limited time, hit these first:

1. **Storage & LVM** ✅ — already done
2. **systemd** — asked in almost every Linux SRE interview
3. **Processes & Signals** — fork/exec/signals are fundamentals
4. **Network Fundamentals** — TCP/IP questions are universal
5. **Troubleshooting Methodology** — the capstone that ties everything
6. **Performance Metrics** — USE method impresses every interviewer
7. **Memory Management** — OOM killer and swap questions are common

---

*Roadmap version 1.0 — built during PERRIO LVM session*
*Update this file after every completed module*
