# Fuel CMS — CTF Write-up

## Target Information

| Information | Details |
|---|---|
| **Target IP** | `10.129.135.241` |
| **Operating System** | Linux |

---

## Enumeration

### Nmap Scan

The first step was to perform a full TCP port scan with service/version detection and default scripts:

```bash
nmap -sC -sV -p- -Pn -oN Ignite.txt 10.129.135.241
```

### Scan Results

```text
Nmap 7.99 scan initiated Tue Aug 25 19:10:02 2026
Nmap scan report for 10.129.135.241

Host is up (0.088s latency).
Not shown: 65534 closed tcp ports (reset)

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
| http-robots.txt: 1 disallowed entry
|_/fuel/
|_http-title: Welcome to FUEL CMS

Service detection performed.
```

### Findings

Only one TCP port was open:

- **Port 80/tcp** → HTTP
- **Web Server** → Apache `2.4.18`
- **OS** → Ubuntu/Linux
- **CMS** → FUEL CMS
- **FUEL CMS version** → `1.4`
- `robots.txt` revealed:

```text
/fuel/
```

### FUEL CMS Enumeration

Navigating to:

```text
http://10.129.135.241/fuel/
```

revealed the **FUEL CMS administration login panel**.

The default credentials were tested:

```text
Username: admin
Password: admin
```

The credentials were valid and provided access to the CMS administration panel.

### SearchSploit

I searched for publicly available exploits related to the identified FUEL CMS version:

```bash
searchsploit fuel cms 1.4
```

An interesting exploit was identified:

```text
47138
```

---

## Exploitation

### FUEL CMS RCE

The identified vulnerability was:

**FUEL CMS 1.4.1 — Remote Code Execution**

```text
CVE-2018-16763
```

The vulnerability allows an attacker to achieve **remote code execution** on a vulnerable FUEL CMS installation.

### Initial Shell

After exploiting the vulnerability, I obtained a web shell on the target.

To transfer additional files to the target, I started a simple HTTP server on my attack machine:

```bash
python -m http.server 8000
```

I then downloaded a PHP reverse shell, modified it with my attack machine's IP address and listening port, and transferred it to the target using `wget`.

On my attack machine, I configured a Netcat listener:

```bash
nc -lvnp 4444
```

After executing the reverse shell on the target, I successfully received a shell connection.

---

# Privilege Escalation

After obtaining a shell as a low-privileged user, I started with the standard manual privilege-escalation checks.

### Sudo Permissions

```bash
sudo -l
```

Nothing useful was found.

### SUID Binaries

I checked for interesting SUID binaries, but nothing immediately useful was identified.

### Cron Jobs

I checked the system's scheduled tasks and cron jobs.

Nothing useful was found.

### Linux Capabilities

I also checked for interesting Linux capabilities.

Nothing useful was identified.

### LinPEAS

Since the manual enumeration did not reveal an obvious privilege-escalation path, I ran **LinPEAS**.

LinPEAS identified a potentially exploitable system component.

The relevant vulnerability was:

```text
CVE-2021-4034 — PwnKit
```

---

## PwnKit — CVE-2021-4034

**CVE-2021-4034**, commonly known as **PwnKit**, is a local privilege-escalation vulnerability affecting `pkexec`, a component of **Polkit**.

The vulnerability can allow an unprivileged local user to escalate privileges to **root** on vulnerable systems.

### Exploitation

The exploit was transferred to the target and compiled:

```bash
gcc exploit.c -o exploit
```

The compiled exploit was then executed.

Successful exploitation resulted in a **root shell**.

```text
CVE-2021-4034 — PwnKit
        ↓
    pkexec / Polkit
        ↓
Local Privilege Escalation
        ↓
      Root Shell
```

---

# Flags

### User Flag

```text
6470e394cbf6dab6a91682cc8585059b
```

### Root Flag

```text
b9bbcb33e11b80be759c4e844862482d
```

---

# Lessons Learned

1. **After obtaining a web shell, always perform a complete privilege-escalation enumeration.**

2. **Start with manual checks** such as:
   - `sudo -l`
   - SUID/SGID binaries
   - Cron jobs
   - Linux capabilities
   - Interesting writable files
   - Running processes
   - Kernel/system information

3. **When manual enumeration does not reveal anything useful, automated tools such as LinPEAS can help identify potential attack paths.**

4. **Kernel and local privilege-escalation vulnerabilities should be considered when other privilege-escalation techniques fail**, while verifying that the exploit matches the target's OS, kernel, architecture, and installed components.

5. This machine demonstrated a common attack chain:

```text
Web Enumeration
      ↓
FUEL CMS Discovery
      ↓
Default Credentials
      ↓
FUEL CMS RCE
      ↓
Web Shell
      ↓
Reverse Shell
      ↓
Privilege Escalation Enumeration
      ↓
LinPEAS
      ↓
CVE-2021-4034 (PwnKit)
      ↓
Root Shell
```

---

# Attack Chain Summary

```text
Port 80
  │
  ├── Apache 2.4.18
  │
  └── FUEL CMS 1.4
          │
          ├── /fuel/ discovered via robots.txt
          │
          ├── Admin panel
          │
          └── admin:admin
                  │
                  ↓
             FUEL CMS RCE
             CVE-2018-16763
                  │
                  ↓
              Web Shell
                  │
                  ↓
           Reverse Shell
                  │
                  ↓
          Privilege Enumeration
                  │
                  ↓
               LinPEAS
                  │
                  ↓
          CVE-2021-4034
              PwnKit
                  │
                  ↓
              Root Shell
```
