---
title: 'byp4ss3d - PICOCTF '
tags: [CTF]

---

---
title: "byp4ss3d - Web Exploitation Writeup"
description:  Directory Traversal,manipulating .htaccess file
tags: CTF,Web,Path Traversal, PortSwigger-2024
robots: index, follow
lang: en
breaks: true
---


# byp4ss3d Writeup

[TOC]

## How it works
The challenge say that we must upload ID student verification and the web only allow the file that only contain file in PNG,JPG or GIF , any file different is not allow

When i send random picture that PNG , the request send to the server its web boundary in the `content_type` where it will store many types of file and Acts as a separator between different form fields and files

=>We can now see that the what type of file called the `Content-Disposition`  for name and filename, `content-type` for type of file

## The Problem

Since challenge ask us to look carefully at how the file upload to server so we can examine it:

:bulb: We know that the server is stored the file we upload the process it in the upload.php path

The only way to actually pass through the upload.php path is try to change it file upload(picture) to give the access to file they want to upload

:point_right: That the idea

## Solution

From the hint from the CTF challenge say that `Apache can be tricked into executing non-PHP files as PHP with a .htaccess file.`
:point_right: This mean that adding a file that have the file name.htaccess(which is path) for directory and then adding php file for acception that make the apache server read the file as the PNG file

Hint 2: `Try uploading more than just one file.
` After give access to server to read the php file as the `image` , we can adding another file that contain php code for execute the code command in the file

### Directory Traversal
Talk a little bit how directory work and how can we exploit it

- By giving the server the file path that access to the `.htaccess` path which can directly to the `Apache` server
-From that we can change the content- type or Content-Disposition from the request and Post to the server

This is how I can change the Request to the server to disguise the php file as image file


![Screenshot 2025-10-12 124923](https://hackmd.io/_uploads/rJItQTOpeg.png)



Ok now go forward and then upload another file contain php code, command the server to execute all the file `ls -la`

This is the second file i upload:




![Screenshot 2025-10-12 124933](https://hackmd.io/_uploads/rkAtX6dpxx.png)






There is a command `system('find / -name "*flag*" 2>/dev/null')` with 2>/dev/null mean you give permission to the server that we will get all the file in type of image but actually it is the php code file. So here is the result

![Screenshot 2025-10-12 125158](https://hackmd.io/_uploads/Byv9XTdTxx.png)

Here we can see the path for the flag `/var/www/flag.txt`



### Final step

Modify your shell.png and get access to the link

![Screenshot 2025-10-12 125445](https://hackmd.io/_uploads/rJlsXp_6xx.png)

**Flag:** Here is the flag `picoCTF{s3rv3r_byp4ss_191e9557}`
**Author:**@minhkhoav47
**solved:** October 12, 2025
