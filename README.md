# SentryLog Security Digest

Automated log auditing and security reporting service for Red Hat Enterprise Linux (RHEL) systems.

---

## Business Case & Problem Statement

IT operations teams managing RHEL servers frequently deal with thousands of daily system journal entries. Manually sifting through logs for authentication failures or critical service errors is time-consuming, error-prone, and delays incident response. Night shift incidents are routinely missed until hours later.

`SentryLog` automates security log analysis by running on a daily schedule, filtering log events via `journalctl` and regular expressions, evaluating source IP failure frequencies, flagging addresses with more than 5 failures as `SUSPECT`, generating dated summary reports, and purging reports older than 30 days.

---

## Architecture & Workflow

1. **Persistent Journaling:** Ensures logs survive system reboots by setting up persistent storage under `/var/log/journal`.
2. **Time Synchronization Fallback:** Configures `chronyd` with `local stratum 10` to maintain time references when outbound NTP (UDP port 123) is blocked by network firewalls.
3. **Log Extraction Engine (`sentrylog.sh`):**
   * Queries logs via `journalctl`.
   * Captures priority levels `err` (3) through `emerg` (0).
   * Isolates SSH authentication failures and service errors (`grep -iE`).
   * Extracts unique IPv4 addresses and assigns operational status (`SUSPECT` vs `NORMAL`).
4. **Scheduled Execution:** Supports systemd timers (`sentrylog.timer`) and cron (`/etc/cron.d/sentrylog`).
5. **Log Retention:** Uses `systemd-tmpfiles` (`/etc/tmpfiles.d/sentrylog.conf`) to clean up reports older than 30 days.

---

## File Architecture

```text
/
├── usr/local/bin/
│   └── sentrylog.sh              # Primary auditing script
├── etc/systemd/system/
│   ├── sentrylog.service         # Systemd service unit (oneshot)
│   └── sentrylog.timer           # Systemd timer unit (daily at 06:00)
├── etc/cron.d/
│   └── sentrylog                 # Daily cron job definition
├── etc/tmpfiles.d/
│   └── sentrylog.conf            # 30-day retention rule
└── var/log/sentrylog/
    └── security_report_*.txt     # Generated report outputs
```

---

## Quick Deployment Guide

### 1. Enable Journal Persistence & Local Time Synchronization

```bash
# Set up persistent journal storage
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald

# Configure local stratum fallback for chrony
echo "local stratum 10" | sudo tee -a /etc/chrony.conf
sudo systemctl restart chronyd
sudo chronyc makestep
```

### 2. Deploy Script Engine

```bash
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
```

### 3. Deploy Systemd Service and Timer

```bash
# Service Unit
sudo bash -c 'cat << "EOF" > /etc/systemd/system/sentrylog.service
[Unit]
Description=SentryLog Security Digest Service
After=network.target chronyd.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/sentrylog.sh
User=root
EOF'

# Timer Unit
sudo bash -c 'cat << "EOF" > /etc/systemd/system/sentrylog.timer
[Unit]
Description=Run SentryLog Security Digest daily at 06:00

[Timer]
OnCalendar=*-*-* 06:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF'

# Enable and start timer
sudo systemctl daemon-reload
sudo systemctl enable --now sentrylog.timer
```

### 4. Deploy Cron Job (Alternative Scheduler)

```bash
sudo bash -c 'cat << "EOF" > /etc/cron.d/sentrylog
0 6 * * * root /usr/local/bin/sentrylog.sh
EOF'

sudo chmod 644 /etc/cron.d/sentrylog
```

### 5. Configure Log Retention Policy

```bash
sudo bash -c 'cat << "EOF" > /etc/tmpfiles.d/sentrylog.conf
d /var/log/sentrylog 0750 root root 30d -
EOF'

sudo systemd-tmpfiles --create /etc/tmpfiles.d/sentrylog.conf
```

---

## Technical Comparison: Systemd Timers vs. Cron

| Feature | Systemd Timers (`sentrylog.timer`) | Cron (`/etc/cron.d/sentrylog`) |
| :--- | :--- | :--- |
| **Service Dependencies** | Supports `After=network.target chronyd.service`. | Executes by clock time without checking service states. |
| **Missed Execution Catch-up** | `Persistent=true` triggers missed runs immediately on boot. | Skips missed runs if system was offline. |
| **Logging & Output** | Native integration with `journalctl -u sentrylog.service`. | Requires MTA setup or manual output redirection. |

---

## Verification & Testing

1. **Inject Test Log Data:**
   ```bash
   # Inject SSH failure events for testing
   for i in {1..7}; do
       logger -t sshd "Failed password for invalid user admin from 192.168.1.50 port 4521$i ssh2"
   done
   ```

2. **Execute Audit Run:**
   ```bash
   sudo /usr/local/bin/sentrylog.sh
   ```

3. **Inspect Generated Report:**
   ```bash
   sudo cat /var/log/sentrylog/security_report_$(date +%Y-%m-%d).txt
   ```

4. **Verify Timer Status:**
   ```bash
   systemctl status sentrylog.timer
   systemctl list-timers | grep sentrylog
   ```
