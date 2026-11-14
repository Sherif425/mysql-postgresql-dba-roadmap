## Lab task: verify disks, firewall ports, journald (Sat 14 Nov 2026)

**Goal:** confirm the three cloned VMs (db1, db2, db3) match the planned layout
before installing any database, then snapshot them as a clean baseline.

### 1. Disks
Commands: `lsblk`, `sudo vgs`, `df -h /data /dblogs /backup`, `cat /etc/fstab`

| VG | Size | Free | Notes |
|---|---|---|---|
| rl_db1 (root) | 55G | 2G | OS volumes: / 15G, /var 8G, /var/lib 10G, /var/log 4G, /var/log/audit 2G, /tmp 4G, /var/tmp 2G, /home 4G |
| vg_data | 243G | 23G | /data 140G, /dblogs 20G, /backup 60G |

- Result: identical on all three VMs. ✅
- **Why separate volumes:** a full binlog/WAL volume (/dblogs) or backup
  volume can't fill the data volume or the OS. The 23G free in vg_data is
  for growing a volume later with `lvextend -r`.
- **Watch out:** packages install data under /var/lib/mysql and
  /var/lib/pgsql by default, and /var/lib is only 10G. In weeks 3–4 move the
  data directories to /data and fix the SELinux labels (`semanage fcontext` +
  `restorecon`).
- **Known quirk:** the root VG is called `rl_db1` on db2 and db3 because they
  were cloned from db1. Harmless, so I'm leaving it. Renaming a root VG
  means updating fstab, GRUB and the initramfs, and a mistake there stops
  the VM booting.

### 2. Firewall ports
`sudo firewall-cmd --permanent --add-port={3306,33060,33061,5432,8008,2379,2380}/tcp && sudo firewall-cmd --reload`

| Port | Service |
|---|---|
| 3306 | MySQL clients |
| 33060 | MySQL X Protocol (Shell, InnoDB Cluster) |
| 33061 | Group Replication between nodes |
| 5432 | PostgreSQL |
| 8008 | Patroni REST API (HAProxy health checks) |
| 2379 / 2380 | etcd client / etcd peer |
| 6632 | SSH (custom port) |

- Result: `--list-ports` shows all of them on all three VMs. ✅
- **Why now:** so replication labs later don't fail because of the firewall.

### 3. journald: what broke and what I learned
- Set `SystemMaxUse=200M` in `/etc/systemd/journald.conf.d/size.conf`.
- `journalctl --disk-usage` showed only 16M. That's normal: the setting is a
  **cap, not a reservation**.
- **Finding:** `journalctl -b -u systemd-journald | grep -i max` showed
  `Runtime Journal (/run/log/journal/…) max 153.6M`.
  - The journal was **volatile**: kept in RAM (/run is tmpfs), so it was
    lost on every reboot.
  - `SystemMaxUse` only applies to a journal on disk (/var/log/journal),
    which didn't exist, so my setting was being ignored.
  - 153.6M was the default runtime limit (10% of /run).
- **Fix:**

[Journal]
Storage=persistent
SystemMaxUse=200M

then `systemctl restart systemd-journald` and `journalctl --flush`.
- **Proof:** the log now says `System Journal (/var/log/journal/…) max 200.0M`,
and after a reboot `journalctl --list-boots` shows more than one boot.
- **Why it matters for a DBA:** after a crash and reboot, the logs that
explain the crash must still be there.

### 4. Clone leftover: old hostname at early boot
- db2's journal showed a `db1` line at the start of the current boot.
The initramfs still contained the old hostname from before cloning.
- Fix: `sudo dracut -f` on db2 and db3, then reboot.

### Lessons
1. Check a setting **took effect**, not just that the file exists.
2. Read the exact words in the log: "Runtime" vs "System" journal told me
 where the logs actually were.
3. Cloned VMs carry leftovers (VG name, hostname in the initramfs, and
 also check machine-id and SSH host keys). List them after every clone.

### Open questions
- Should /var/lib be grown, or is moving the data directories to /data enough? (Ask the mentor.)

### Time
Planned 0.5 h · Actual: 2 h