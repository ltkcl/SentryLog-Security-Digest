# RHEL Systems Administration Projects

Automated security log auditing and storage management tools for Red Hat Enterprise Linux (RHEL) systems. This repository contains technical runbooks, shell scripts, and configuration files for two enterprise system administration utilities: **SentryLog Security Digest** and **DiskDetective Storage Audit**.

---

## Repository Structure

```text
.
├── README.md
├── sentrylog/
│   ├── bin/
│   │   └── sentrylog.sh
│   ├── systemd/
│   │   ├── sentrylog.service
│   │   └── sentrylog.timer
│   ├── cron/
│   │   └── sentrylog
│   └── tmpfiles/
│       └── sentrylog.conf
└── disk_detective/
    ├── bin/
    │   └── disk_detective.sh
    └── docs/
        └── link_investigation.md
```

---

## Project 1: SentryLog Security Digest

### Business Case & Problem Statement
IT operations teams managing RHEL servers often deal with thousands of daily journal entries. Manually reviewing logs for authentication failures or critical service errors is time-consuming and delays incident response. 

`SentryLog` automates log analysis by running on a daily schedule, filtering log events via `journalctl` and regular expressions, evaluating source IP failure frequencies, flagging addresses with more than 5 failures as `SUSPECT`, generating dated summary reports, and automatically purging old reports after 30 days.

### Mechanics & Workflow
1. **Persistent Journaling:** Ensures logs survive system reboots under `/var/log/journal`.
2. **Time Sync Fallback:** Configures `chronyd` with `local stratum 10` to maintain time references when outbound NTP (UDP 123) is blocked by network firewalls.
3. **Log Engine (`sentrylog.sh`):**
   * Parses logs via `journalctl`.
   * Captures priority levels `err` (3) through `emerg` (0).
   * Isolates SSH authentication failures and service errors (`grep -iE`).
   * Extracts unique IPv4 addresses and assigns operational status (`SUSPECT` vs `NORMAL`).
4. **Dual Scheduling:** Supports systemd timers (`sentrylog.timer`) and cron (`/etc/cron.d/sentrylog`).
5. **Log Retention:** Uses `systemd-tmpfiles` (`/etc/tmpfiles.d/sentrylog.conf`) to remove reports older than 30 days.

### Quick Setup: SentryLog

```bash
# 1. Enable Journal Persistence & Local Time Fallback
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald
echo "local stratum 10" | sudo tee -a /etc/chrony.conf
sudo systemctl restart chronyd
sudo chronyc makestep

# 2. Deploy Script Engine
sudo mkdir -p /var/log/sentrylog
sudo bash -c 'cat << "EOF" > /usr/local/bin/sentrylog.sh
#!/bin/bash

LOG_DIR="/var/log/sentrylog"
OUT="${LOG_DIR}/security_report_$(date +%Y-%m-%d).txt"

mkdir -p "$LOG_DIR"
exec > "$OUT" 2>&1

echo "=================================================="
echo "SENTRYLOG SECURITY DIGEST"
echo "Generated: $(date)"
echo "Host: $(hostname)"
echo "=================================================="
echo ""

echo "--- TIME SYNC ---"
chronyc tracking | head -n 5
echo ""

echo "--- EVENT COUNTS ---"
ERR_CNT=$(journalctl -p err..emerg -b --no-pager 2>/dev/null | wc -l)
FAIL_CNT=$(journalctl -b --no-pager 2>/dev/null | grep -iE "failed|failure" | wc -l)
AUTH_LOGS=$(journalctl -u sshd -b --no-pager 2>/dev/null | grep -iE "failed password|authentication failure")

if [ -z "$AUTH_LOGS" ]; then
    AUTH_CNT=0
else
    AUTH_CNT=$(echo "$AUTH_LOGS" | wc -l)
fi

echo "Priority Errors (err+): $ERR_CNT"
echo "Service Failures: $FAIL_CNT"
echo "Failed Authentications: $AUTH_CNT"
echo ""

echo "--- IP ANALYSIS ---"
if [ -z "$AUTH_LOGS" ]; then
    echo "No failed login attempts found."
else
    IP_LIST=$(echo "$AUTH_LOGS" | grep -oE "([0-9]{1,3}\.){3}[0-9]{1,3}" | sort -u)
    for ip in $IP_LIST; do
        cnt=$(echo "$AUTH_LOGS" | grep -c "$ip")
        status="NORMAL"
        [ "$cnt" -gt 5 ] && status="SUSPECT"
        echo "IP: $ip | Count: $cnt | Status: $status"
    done
fi

chmod 640 "$OUT"
EOF'
sudo chmod +x /usr/local/bin/sentrylog.sh

# 3. Deploy Systemd Timer (Primary Scheduler)
sudo bash -c 'cat << "EOF" > /etc/systemd/system/sentrylog.service
[Unit]
Description=SentryLog Security Digest Service
After=network.target chronyd.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/sentrylog.sh
User=root
EOF'

sudo bash -c 'cat << "EOF" > /etc/systemd/system/sentrylog.timer
[Unit]
Description=Run SentryLog Security Digest daily at 06:00

[Timer]
OnCalendar=*-*-* 06:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF'

sudo systemctl daemon-reload
sudo systemctl enable --now sentrylog.timer

# 4. Configure Retention Rule (30 Days)
sudo bash -c 'cat << "EOF" > /etc/tmpfiles.d/sentrylog.conf
d /var/log/sentrylog 0750 root root 30d -
EOF'
sudo systemd-tmpfiles --create /etc/tmpfiles.d/sentrylog.conf
```

---

## Project 2: DiskDetective Storage Audit

### Business Case & Problem Statement
A shared RHEL file server hosting video assets reaches critical capacity (96% utilization), preventing users from saving new renders. Unindexed project files, temporary render files, and stale data consume active space without visibility.

`DiskDetective` provides an automated storage auditing script that surveys filesystem utilization, scans for files over 100 MB, isolates stale files unmodified for over 180 days, calculates reclaimable space, generates executive summaries, and offloads stale data to secondary archive storage.

### Mechanics & Workflow
1. **Survey:** Uses `lsblk`, `df -h`, and `du` to isolate mounted filesystems and directory size consumption.
2. **Targeted Discovery (`find`):**
   * Identifies large files (>100 MB).
   * Identifies stale files (>180 days unmodified).
   * Filters files by owner (`root` or unassigned users).
3. **Links Mechanics Investigation:** Analyzes inode references, link counts, and space release behaviors for hard links vs. symbolic links.
4. **Automated Audit Pipeline (`disk_detective.sh`):** Outputs consolidated metrics to `/var/log/storage_audit.txt`.
5. **Cold Storage Offloading:** Mounts a virtual archive volume (`/mnt/cold_archive`) and moves stale assets out of primary storage.

### Quick Setup: DiskDetective

```bash
# 1. Deploy DiskDetective Script Engine
sudo bash -c 'cat << "EOF" > /usr/local/bin/disk_detective.sh
#!/bin/bash

REPORT="/var/log/storage_audit.txt"

{
    echo "=================================================="
    echo "         DISKDETECTIVE STORAGE AUDIT REPORT       "
    echo "         Generated: $(date)                       "
    echo "         Host: $(hostname)                        "
    echo "=================================================="
    echo ""

    echo "--- 1. FILESYSTEM PRESSURE SURVEY ---"
    df -h / | awk "NR==1 || NR==2"
    echo ""

    echo "--- 2. TOP 10 LARGEST FILES (>100MB) ---"
    find / -type f -size +100M -exec du -h {} + 2>/dev/null | sort -rh | head -n 10
    echo ""

    echo "--- 3. STALE FILES UNMODIFIED IN >180 DAYS ---"
    find / -type f -mtime +180 -exec du -h {} + 2>/dev/null | sort -rh | head -n 20
    echo ""

    echo "--- 4. RECLAIMABLE SPACE SUMMARY ---"
    STALE_COUNT=$(find / -type f -mtime +180 2>/dev/null | wc -l)
    STALE_SIZE=$(find / -type f -mtime +180 -exec du -ch {} + 2>/dev/null | grep total$ | awk "{print \$1}")
    [ -z "$STALE_SIZE" ] && STALE_SIZE="0B"
    
    echo "Total Stale Files Identified: $STALE_COUNT"
    echo "Total Reclaimable Disk Space: $STALE_SIZE"
    echo ""

    echo "=================================================="
    echo "              END OF STORAGE AUDIT                "
    echo "=================================================="
} > "$REPORT"

chmod 644 "$REPORT"
EOF'

sudo chmod +x /usr/local/bin/disk_detective.sh

# 2. Execute Audit Run
sudo /usr/local/bin/disk_detective.sh
sudo cat /var/log/storage_audit.txt

# 3. Cold Storage Setup and Offload
sudo mkdir -p /mnt/cold_archive
sudo dd if=/dev/zero of=/opt/cold_storage.img bs=1M count=500
sudo mkfs.ext4 /opt/cold_storage.img
sudo mount -o loop /opt/cold_storage.img /mnt/cold_archive

sudo mkdir -p /mnt/cold_archive/stale_files_archive
sudo find / -type f -mtime +180 -name "*.tmp" -exec cp --parents {} /mnt/cold_archive/stale_files_archive/ \; 2>/dev/null
```

---

## Technical Comparison: Systemd Timers vs. Cron

| Feature | Systemd Timers (`sentrylog.timer`) | Cron (`/etc/cron.d/sentrylog`) |
| :--- | :--- | :--- |
| **Service Dependencies** | Supports `After=network.target chronyd.service`. | Runs strictly by clock time without checking services. |
| **Missed Job Catch-up** | `Persistent=true` executes missed runs on boot. | Skips missed runs if system was offline. |
| **Logging & Output** | Native integration with `journalctl -u sentrylog.service`. | Requires MTA setup or explicit stdout redirection. |

---

## Hard Links vs. Symbolic Links Summary

* **Hard Link:** Points directly to the inode address on disk. Deleting the original file decrements the link count by 1. The underlying data remains intact and disk space is **not released** until all hard links referencing that inode are deleted.
* **Symbolic Link:** Points to the target file path. Deleting the original file leaves the soft link broken (`No such file or directory`). Disk space occupied by the source file is released immediately upon deletion (provided no hard links exist).
