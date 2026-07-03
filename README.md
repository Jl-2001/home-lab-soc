# Home Lab SOC — Penetration Testing & SIEM Detection

A full offensive + defensive home lab built on VirtualBox, documenting a complete attack → detect → prevent pipeline.

## Lab Environment

| Host | IP | Role |
|------|----|------|
| Ubuntu 24.04 (host) | 192.168.1.2 | SIEM (Splunk Enterprise) |
| pfSense | 192.168.1.1 / 192.168.2.1 | Firewall / Router |
| Kali Linux | 192.168.1.100 | Attacker |
| Metasploitable 2 | 192.168.2.10 | Target |

The lab evolved from a single flat host-only network (vboxnet0) into a segmented topology behind a pfSense firewall, with Kali on a LAN segment and Metasploitable isolated on its own network (MetasploitableNet), reflecting a more realistic attacker/target separation.

## Lab 1 — Penetration Test

- Reconnaissance with Nmap (`nmap -sV`)
- Exploited vsftpd 2.3.4 backdoor (CVE-2011-2523) via Metasploit (`exploit/unix/ftp/vsftpd_234_backdoor`)
- Gained root shell, discovered `+ +` misconfiguration in `/root/.rhosts`
- Extracted `/etc/shadow` via Meterpreter
- Cracked 6/7 password hashes with John the Ripper (rockyou.txt + custom wordlist)
- Captured plaintext Telnet credentials via Wireshark TCP stream reconstruction

[View Pentest Documentation](./pentest/)

## Lab 2 — SIEM Detection with Splunk

- Installed Splunk Enterprise on the Ubuntu host
- Configured syslog forwarding from Kali and Metasploitable
- Simulated an SSH brute-force attack from Kali → Metasploitable
- Built a detection rule: alert fires when more than 5 failed SSH logins occur within a 5-minute window
- Built a SOC dashboard with attack timeline, top attacker IPs, and total failed login count

[View SIEM Documentation](./siem/)

## Lab 3 — Network Segmentation & Firewall Enforcement

- Deployed pfSense as a router/firewall between attacker and target
- Segmented the network: Kali on LAN (192.168.1.0/24), Metasploitable isolated on its own interface (MetasploitableNet, 192.168.2.0/24)
- Diagnosed and resolved a firewall rule that initially failed silently for three separate reasons:
  1. The rule was disabled in the pfSense UI
  2. Kali and Metasploitable were on the same Layer 2 segment, so traffic never actually routed through the firewall to be evaluated
  3. After segmenting the networks, a stale DHCP-assigned secondary IP on Kali didn't match the rule's configured source address
- Created a working block rule denying SSH (TCP/22) from Kali to Metasploitable, confirmed via `nc` connection timeouts and pfSense firewall logs
- Forwarded pfSense firewall logs to Splunk via syslog (UDP 514)
- Built a custom Splunk field extraction to parse pfSense's `filterlog` format into structured fields (`src_ip`, `dest_ip`, `src_port`, `dest_port`, `protocol`, `fw_action`)
- Built a second SOC dashboard ("Firewall Segmentation Enforcement") visualizing blocked connection attempts

[View Firewall/Segmentation Screenshots](./siem/screenshots/)

## Tools Used

Splunk Enterprise · pfSense · Metasploit · John the Ripper · Wireshark · Nmap · Kali Linux · Metasploitable 2 · VirtualBox
