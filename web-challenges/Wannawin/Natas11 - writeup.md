---
title: Natas11 - writeup
tags: [CTF, ' Web', ' SQL Injection']

---

---
title: "Natas 10 -> 11:Overthewire"
description: XOR encryption from cyphertext to plaintext
tags: CTF, Web, XOR-encryption
robots: index, follow
lang: en
breaks: true
---

# Natas11 Writeup

[TOC]

## How it works
this web design just like the same to the level 10 but the password being protected with the XOR_encryption

From the request, we could see there is a cookie data for web application and when we access to the web page there is the background color for set color but it does not matter the result

The thing that we must focus on is the set_cookie data in response

## problem
Look  at the viewsource code, we could see this is the blended code with html and php, `index-source.html` , i can see there are conditions to get the flag:

```php=
$defaultdata = array( "showpassword"=>"no", "bgcolor"=>"#ffffff");

function xor_encrypt($in) {
    $key = '<censored>';
    $text = $in;
    $outText = '';

    // Iterate through each character
    for($i=0;$i<strlen($text);$i++) {
    $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }

    return $outText;
}

```
this one is for the request when we have our cookie and the cookie is being encrypted with XOR for each `$i`

Another condition is that 
```php=
function loadData($def) {
    global $_COOKIE;
    $mydata = $def;
    if(array_key_exists("data", $_COOKIE)) {
    $tempdata = json_decode(xor_encrypt(base64_decode($_COOKIE["data"])), true);
    if(is_array($tempdata) && array_key_exists("showpassword", $tempdata) && array_key_exists("bgcolor", $tempdata)) {
        if (preg_match('/^#(?:[a-f\d]{6})$/i', $tempdata['bgcolor'])) {
        $mydata['showpassword'] = $tempdata['showpassword'];
        $mydata['bgcolor'] = $tempdata['bgcolor'];
        }
    }
    }
    return $mydata;
}


```
This mean that it will edit the cookie and save it to `$tempdata` with json_decode -> XOR_encrypt -> base64_decode 

The php code also block all the special syntax like `'/^#(?:[a-f\d]`

Finally the `$tempdata` being encoded to become another plaintext being encrypted with `cyphertext` and `key` and it must pass this condition to pass the text
```php=
function saveData($d) {
    setcookie("data", base64_encode(xor_encrypt(json_encode($d))));
}

$data = loadData($defaultdata);

if(array_key_exists("bgcolor",$_REQUEST)) {
    if (preg_match('/^#(?:[a-f\d]{6})$/i', $_REQUEST['bgcolor'])) {
        $data['bgcolor'] = $_REQUEST['bgcolor'];
    }
}

saveData($data);
```
See that, the cookie being encoded again 

## solution

### Step 1
So the first thought that come up to my mind is that we must write up our script for the find the key of the cookie

because the cookie being encrypted not only 1 but 2 times

When I find the key , I could decrypted the cyphertext to find the real plaintext of `newcookie`


Here is my first code for finding the key


```php=
#!/usr/bin/php
<?php
$defaultdata = array("showpassword"=>"no", "bgcolor"=>"#ffffff");

function xor_encrypt($in) {
        $key = base64_decode('HmYkBwozJw4WNyAAFyB1VUcqOE1JZjUIBis7ABdmbU1GIjEJAyIxTRg%3D');
        $text = $in;
        $outText = '';

        for($i=0;$i<strlen($text); $i++){
        $outText .= $text[$i] ^ $key[$i % strlen($key)];
        }
        return $outText;
}
$b = xor_encrypt(json_encode($defaultdata));
print($b);
?>


```

Next you will give permission for the code running and get the key`chmod +x sample.php                                                                        
./sample.php`
here is the key `eDWoeDWoeDWoeDWoeDWoeDWoeDWoeDWoeDWoeDWoe`

### Step2
You will create another find for check the condition for password checking if yes
 here is the code
```php=
#!/usr/bin/php
<?php
$defaultdata = array( "showpassword"=>"yes", "bgcolor"=>"#ffffff");
function xor_encrypt($in){
        $key = "eDWo";
        $text = $in;
        $outText = '';

        for($i=0;$i<strlen($text);$i++){
        $outText .= $text[$i] ^ $key[$i % strlen($key)];
        }
        return $outText;
}
$new_cookie = base64_encode(xor_encrypt(json_encode($defaultdata)));
print($new_cookie);
?>
```


for this code it will get the key for decrytion from the `cyphertext` and we will get out new_cookie back (the correct one)
here is the new-cookie:
```
HmYkBwozJw4WNyAAFyB1VUc9MhxHaHUNAic4Awo2dVVHZzEJAyIxCUc5
```

### Final
Get access to burp suite and go back to the link again

From the `request` , change the old cookie to the new_cookie and pick forward, and your flag is there

**Flag** `The password for natas12 is yZdkjAYZRd3R7tq7T5kXMjMJlOIkzDeB`

**Author:** @minhkhoav47  
**Solved:** October 10, 2025

![image](https://hackmd.io/_uploads/SJSu9yMvze.png)
![image](https://hackmd.io/_uploads/r1fSokMDGg.png)




