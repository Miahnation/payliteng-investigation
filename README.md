# PayliteNG Security Investigation 🔍

> A complete, end-to-end cybersecurity investigation of a fictional Nigerian fintech breach —
> covering attack reconstruction, multi-layer detection, digital forensics, and incident response.

![Status](https://img.shields.io/badge/Status-In%20Progress-orange)
![Tools](https://img.shields.io/badge/Tools-Splunk%20%7C%20Wazuh%20%7C%20Snort%203.0%20%7C%20Suricata%20%7C%20Volatility%203-blue)
![Context](https://img.shields.io/badge/Context-Nigerian%20Fintech-green)

---

## The Scenario

**Company:** PayliteNG Limited — a Nigerian fintech based in Aba, Abia State  
**Customers:** 200,000 registered users  
**Stack:** PHP/MySQL · Apache · vsftpd · Linux  
**Date of breach:** January 13, 2025  
**Discovered:** January 20, 2025 — 7 days after the attack  

On the night of January 13th, an attacker identifying as **@gr4yh4t** launched a systematic attack against PayliteNG's production server. By 03:52 AM, they had root access. By 03:51:48, 45.6MB of customer data was on its way out.

This repository documents the complete investigation — from raw log analysis to memory forensics — and builds the detection stack that would have caught the attack within seconds.

---

## Attack Chain

```
03:47:15  SSH brute force begins — 47 attempts in 19 seconds
          Source: 185.220.101.47 (Tor exit node)

03:48:01  Web server reconnaissance — scanning for exposed paths

03:51:44  Anonymous FTP login — no password required
          vsftpd 2.3.4 — CVE-2011-2523 — CVSS 10.0

03:51:48  customer_backup_2024.sql.gz downloaded — 45.6MB
          200,000 customer records exfiltrated

03:52:00  vsftpd backdoor triggered — root shell obtained

03:53:00  /etc/shadow read — all password hashes stolen

03:54:00  Cron persistence installed — callback every 5 minutes

09:15:44  Second attacker (203.0.113.88) — SQL injection on login.php
```

---

## MITRE ATT&CK Mapping

| Technique ID | Name | Evidence |
|---|---|---|
| T1110.001 | Password Guessing | 47 SSH failures in auth.log |
| T1190 | Exploit Public-Facing Application | vsftpd CVE-2011-2523 |
| T1078 | Valid Accounts (Anonymous FTP) | vsftpd anonymous login |
| T1005 | Data from Local System | 45.6MB customer DB downloaded |
| T1048.003 | Exfiltration Over Unencrypted Protocol | FTP plaintext transfer |
| T1053.003 | Cron | Persistence via /etc/crontab |
| T1071 | Application Layer Protocol | curl C2 callback |
| T1003.008 | /etc/shadow | Credential access |

---

## Detection Coverage

| Attack | Splunk | Wazuh | Snort 3.0 | Suricata |
|---|---|---|---|---|
| SSH brute force | ✅ | ✅ | ✅ | ✅ |
| Anonymous FTP | ✅ | ❌ | ✅ | ✅ |
| Data exfiltration | ✅ | ❌ | ✅ | ✅ |
| SQL injection | ✅ | ❌ | ✅ | ✅ |
| File modification | ❌ | ✅ | ❌ | ❌ |
| Cron backdoor | ❌ | ✅ | ❌ | ❌ |
| C2 callback | ❌ | ❌ | ✅ | ✅ |

---

## Repository Structure

```
payliteng-investigation/
├── 01-logs/
│   ├── auth.log               ← SSH brute force evidence
│   ├── ftp.log                ← Anonymous FTP evidence
│   └── apache_access.log      ← SQL injection evidence
│
├── 02-detection/
│   ├── splunk/
│   │   └── detection_rules.spl    ← 5 SPL rules
│   ├── snort3/
│   │   └── payliteng.rules        ← 5 Snort 3.0 rules
│   └── suricata/
│       └── payliteng.rules        ← 5 Suricata rules
│
├── 03-forensics/
│   ├── disk/
│   │   ├── forensics_commands.md  ← TSK commands used
│   │   └── recovered_backdoor.sh  ← Deleted file recovered
│   └── memory/
│       ├── volatility_commands.md ← Volatility 3 plugins used
│       └── memory_findings.md     ← What was found in RAM
│
├── 04-attack-mapping/
│   └── payliteng_attck_layer.json ← ATT&CK Navigator layer
│
├── 05-reports/
│   ├── incident_report.md         ← Full IR report
│   └── chain_of_custody.md        ← Evidence log
│
└── README.md
```

---

## Detection Rules Included

### Snort 3.0 Rules
```
alert tcp any any -> $HOME_NET 22 (msg:"PAYLITENG SSH Brute Force"; flow:to_server; detection_filter:track by_src, count 5, seconds 60; sid:1000002; rev:1;)

alert tcp any any -> $HOME_NET 21 (msg:"PAYLITENG Anonymous FTP Login"; flow:to_server,established; content:"USER anonymous"; nocase; sid:1000003; rev:1;)

alert http any any -> any any (msg:"PAYLITENG SQL Injection UNION SELECT"; flow:to_server,established; http.uri; content:"UNION"; nocase; content:"SELECT"; nocase; sid:1000004; rev:1;)
```

### Splunk SPL Rules
```spl
# SSH Brute Force Detection
index=payliteng "Failed password"
| rex "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats count by src_ip
| where count > 5
| sort -count
```

---

## Regulatory Impact

| Requirement | Deadline | Status |
|---|---|---|
| NDPC Breach Notification | 72 hours from discovery | MISSED — discovered 7 days late |
| CBN Incident Report | Immediate | OVERDUE |
| Potential NDPC fine | — | Up to 2% annual revenue or ₦10 million |

**Root cause of missed deadline:** No SIEM monitoring. Attack ran for 7 days undetected.  
**With Splunk running:** Alert fires at 03:47:20 — 72-hour NDPC clock starts on time.

---

## Tools Used

- **Log Analysis:** grep, awk, sed, Splunk SPL
- **Network Detection:** Snort 3.0, Suricata, tcpdump, Wireshark
- **Host Detection:** Wazuh HIDS
- **Disk Forensics:** The Sleuth Kit (fls, icat, mactime), Autopsy
- **Memory Forensics:** Volatility 3, LiME
- **Threat Intelligence:** MITRE ATT&CK Navigator, AbuseIPDB
- **Framework:** NIST SP 800-61 Incident Response

---

## About This Project

This investigation was developed as the backbone of the **TechRise 3.0 Cybersecurity Programme** in Aba, Nigeria. The PayliteNG scenario ran as a continuous thread across all 12 weeks — connecting every lab to a realistic Nigerian fintech breach that students could relate to.

*Part of the [30-Day Post-Cohort Challenge](#)*

---

**Author:** Akorita-Ifeanyichukwu Nehemiah  
**LinkedIn:** [Connect](www.linkedin.com/in/akorita-nehemiah-21aab8223) · **Blog:** [Read](https://www.blogger.com/blog/posts/3275027532616772935) · **Twitter:** [Follow](https://x.com/Nehemia17303777)
[Read the full case study →](CASE_STUDY.md)
