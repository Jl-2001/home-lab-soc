# SIEM Detection Lab — Splunk

Configured Splunk Enterprise as a SIEM to ingest logs from lab hosts, detect a simulated SSH brute-force attack, and visualize the results on a SOC-style dashboard.

## Environment

| Host | IP | Role |
|------|----|------|
| Ubuntu 24.04 (host) | 192.168.56.1 (later 192.168.1.2) | SIEM (Splunk Enterprise) |
| Kali Linux | 192.168.56.102 (later 192.168.1.100) | Attacker |
| Metasploitable 2 | 192.168.56.101 (later 192.168.2.10) | Target |

## 1. Splunk Installation

Installed Splunk Enterprise on the Ubuntu host via `.deb` package:

```bash
sudo dpkg -i splunk-*.deb
sudo /opt/splunk/bin/splunk start --run-as-root
```

Configured a UDP input on port 514 to receive syslog data from lab hosts (**Settings → Data Inputs → UDP → New**, port 514, source type `syslog`).

## 2. Log Forwarding Setup

**Kali (rsyslog):**
```bash
sudo apt install rsyslog
```
Configured to forward logs to the Splunk host's syslog listener.

**Metasploitable 2 (sysklogd):**

Metasploitable's older `sysklogd` daemon doesn't support the standard `@host:port` syntax — the working config in `/etc/syslog.conf` required:
```
*.* @192.168.56.1
```
(port 514 implied). After correcting this and restarting sysklogd, Splunk began receiving live events from both hosts.

## 3. Brute Force Detection

Simulated an SSH brute-force attack from Kali against Metasploitable:

```bash
for i in {1..10}; do ssh -o StrictHostKeyChecking=no wronguser@192.168.56.101; done
```

Configured a scheduled Splunk alert, **"SSH Brute Force Detected"**, using a search that counts failed SSH login events within a rolling window and fires when the threshold is exceeded (more than 5 failed logins within 5 minutes), scheduled with cron expression `*/5 * * * *`.

The alert fired successfully during testing, confirming end-to-end detection: attack traffic → log ingestion → correlation search → alert.

## 4. SOC Dashboard

Built a three-panel dashboard, **"SOC Home Lab - Brute Force Monitor"**:

1. **Line chart** — failed SSH logins over time
2. **Bar chart** — top attacking IPs (Kali's address led with the highest attempt count)
3. **Single value panel** — total failed login count

## Key Takeaways

- Centralized logging turns scattered host-level events into correlatable, searchable data.
- Legacy syslog daemons (like `sysklogd` on older Linux distros) can have subtly different forwarding syntax than modern `rsyslog`/`syslog-ng` — worth checking before assuming a config is broken.
- A simple threshold-based correlation search is enough to catch a basic brute-force pattern; more advanced detections (e.g. accounting for distributed/low-and-slow attacks) would need additional tuning.
- Dashboards turn raw detection into something a SOC analyst can actually monitor at a glance.

## Related Work

See [Lab 3 — Network Segmentation & Firewall Enforcement](../README.md#lab-3--network-segmentation--firewall-enforcement) for the follow-on work extending this Splunk instance to ingest and visualize pfSense firewall logs.
