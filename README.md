# SentryLog-Security-Digest
SentryLog Security DigestAn automated security auditing and log analysis tool for Red Hat Enterprise Linux (RHEL) systems. SentryLog parses system logs (journalctl), extracts priority errors and SSH authentication failures, calculates source IP failure frequencies, flags suspect origin addresses, and generates scheduled executive digests.Table of ContentsBusiness Case & OverviewArchitecture & MechanicsRepository Structure & PathsPrerequisitesInstallation & SetupUsage & ExecutionSystemd Timer vs CronTesting & VerificationReport Retention & CleanupBusiness Case & OverviewIT operations teams managing RHEL servers often deal with thousands of system journal entries daily. Manually reviewing logs for failed login attempts or priority service errors is inefficient and delays incident response, allowing overnight security events to go unnoticed for hours.SentryLog automates log review by executing on a daily schedule, filtering critical log events via journalctl and regular expressions, evaluating source IP failure frequencies (flagging IPs with > 5 failures as SUSPECT), writing dated report summaries, and automatically purging old reports after 30 days.Architecture & MechanicsPersistent Journaling: Logs are configured to survive system reboots under /var/log/journal.Clock Synchronization Fallback: Configures chronyd with local stratum 10 to maintain internal time references when outbound NTP (UDP 123) is blocked by network firewalls.Log Engine (sentrylog.sh):Parses system logs using journalctl.Captures priority levels err (Level 3) through emerg (Level 0).Isolates SSH authentication failures and service errors (grep -iE).Extracts unique IPv4 addresses and assigns operational status:Failures > 5: Marked as SUSPECTFailures <= 5: Marked as NORMALDual Scheduler Support: Includes systemd timer (sentrylog.timer) and cron (/etc/cron.d/sentrylog) configurations.Log Retention: Uses systemd-tmpfiles (/etc/tmpfiles.d/sentrylog.conf) to remove reports older than 30 days.Repository Structure & PathsSentryLog-Security-Digest/
├── README.md
├── bin/
│   └── sentrylog.sh
├── systemd/
│   ├── sentrylog.service
│   └── sentrylog.timer
├── cron/
│   └── sentrylog
└── tmpfiles/
    └── sentrylog.conf
System Installation Target Paths:/usr/local/bin/sentrylog.sh/etc/systemd/system/sentrylog.service/etc/systemd/system/sentrylog.timer/etc/cron.d/sentrylog/etc/tmpfiles.d/sentrylog.conf/var/log/sentrylog/ (Output directory)PrerequisitesOS: RHEL 8/9, CentOS Stream, or RHEL binary compatible distributions.Privileges: root or sudo access.Utilities: systemd-journald, chrony, bash, grep, coreutils.Installation & Setup1. Enable Journal Persistence & Chrony Fallback# Enable journal persistence
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald

# Set local stratum fallback for chrony
echo "local stratum 10" | sudo tee -a /etc/chrony.conf
sudo systemctl restart chronyd
sudo chronyc makestep
2. Deploy Script Enginesudo mkdir -p /var/log/sentrylog

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
3. Deploy Systemd Timer (Primary Scheduler)# Service Unit
sudo bash -c 'cat << "EOF" > /etc/systemd/system/sentrylog.service
[Unit]
Description=SentryLog Security Digest Service
After=network.target chronyd.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/sentrylog.sh
User=root
EOF'

# Timer Unit (Daily at 06:00)
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
4. Deploy Cron Job (Alternative Scheduler)sudo bash -c 'cat << "EOF" > /etc/cron.d/sentrylog
0 6 * * * root /usr/local/bin/sentrylog.sh
EOF'

sudo chmod 644 /etc/cron.d/sentrylog
5. Configure Report Retention Rulesudo bash -c 'cat << "EOF" > /etc/tmpfiles.d/sentrylog.conf
# Type Path                 Mode User Group Age Argument
d     /var/log/sentrylog   0750 root root  30d  -
EOF'

sudo systemd-tmpfiles --create /etc/tmpfiles.d/sentrylog.conf
Usage & ExecutionManual Runsudo /usr/local/bin/sentrylog.sh
Viewing Generated Reportssudo cat /var/log/sentrylog/security_report_$(date +%Y-%m-%d).txt
Sample Output:==================================================
SENTRYLOG SECURITY DIGEST
Generated: Sun Sep 20 06:00:00 EDT 2026
Host: rhel-server01
==================================================

--- TIME SYNC ---
Reference ID    : 7F7F0101 (LOCAL)
Stratum         : 10
Ref time (UTC)  : Sun Sep 20 10:00:00 2026
System time     : 0.000000000 seconds slow of NTP time
Last offset     : +0.000000000 seconds

--- EVENT COUNTS ---
Priority Errors (err+): 12
Service Failures: 4
Failed Authentications: 9

--- IP ANALYSIS ---
IP: 10.0.0.15 | Count: 2 | Status: NORMAL
IP: 192.168.1.50 | Count: 7 | Status: SUSPECT

==================================================
Systemd Timer vs CronFeatureSystemd Timers (sentrylog.timer)Cron (/etc/cron.d/sentrylog)DependenciesSupports After=network.target chronyd.service.Executes strictly by system clock time.Missed Job Catch-upPersistent=true runs missed jobs on boot.Skips missed runs if system was offline.LoggingNative logging via journalctl -u sentrylog.service.Requires MTA or manual file redirection.Testing & VerificationSimulate authentication failure entries using logger to test threshold logic:# Inject 7 failure events for IP 192.168.1.50 (Triggers SUSPECT status)
for i in {1..7}; do
    logger -t sshd "Failed password for invalid user admin from 192.168.1.50 port 4521$i ssh2"
done

# Inject 2 failure events for IP 10.0.0.15 (Triggers NORMAL status)
for i in {1..2}; do
    logger -t sshd "Failed password for root from 10.0.0.15 port 3310$i ssh2"
done

# Run script and verify output
sudo /usr/local/bin/sentrylog.sh
sudo cat /var/log/sentrylog/security_report_$(date +%Y-%m-%d).txt
Verify systemd timer state:systemctl status sentrylog.timer
systemctl list-timers | grep sentrylog
Report Retention & CleanupOld reports are cleaned up according to /etc/tmpfiles.d/sentrylog.conf. To manually run or test the cleanup process:sudo systemd-tmpfiles --clean /etc/tmpfiles.d/sentrylog.conf
