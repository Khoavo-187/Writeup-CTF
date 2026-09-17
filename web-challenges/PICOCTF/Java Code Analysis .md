---
title: 'Java Code Analysis '
tags: [' Web', CTF]

---

---
title: "Java Code Analysis - Web exploitation Writeup"
description: JWT Token vulnerability in cracking the secret key
tags: CTF, JWT-Token, picoCTF
robots: index,follow
breaks: true
---


# JAVA code Writeup

[TOC]

## How it works

First, this is the bookshelf website where you can search for your book that you want to read

the server give the different `Authorization` for every user, but the one that could not be modified is the admin user

In the bookshelf website, there is the flag just for the `admin` authorization

==> This is the main point to get the flag, you must log in as admin user

## The problem

For finding the vulnerbility, the website give you a zip contains many files in `Java`

So basically this is the website coded by Java and using the JWT token for authorize every users log in the website 

There are 3 main directory that can be found very useful

1. Thebookshelfconfig.java

``` java

User freeUser = new User();
                freeUser.setProfilePicName("default-avatar.png")
                        .setRole(FreeRole)
                        .setLastLogin(LocalDateTime.now())
                        .setFullName("User")
                        .setEmail("user")
                        .setPassword(passwordEncoder.encode("user"));
                userRepository.save(freeUser);

                User admin = new User();
                admin.setProfilePicName("default-avatar.png")
                        .setRole(AdminRole)
                        .setLastLogin(LocalDateTime.now())
                        .setFullName("Admin")
                        .setEmail("admin")
                        .setPassword(passwordEncoder.encode("<redacted>"));
                userRepository.save(admin);

                logger.info("initialized 'admin' and 'user' users.");
```

this mean that if you want to get the admin access, you must have token payload for `userId` , `email` or `name` that have the admin inform

2. SecretGenerator.java

``` java
private UserDataPaths userDataPaths;

    private String generateRandomString(int len) {
        // not so random
        return "1234";
    }

    String getServerSecret() {
        try {
            String secret = new String(FileOperation.readFile(userDataPaths.getCurrentJarPath(), SERVER_SECRET_FILENAME), Charset.defaultCharset());
            logger.info("Server secret successfully read from the filesystem. Using the same for this runtime.");
            return secret;
        }catch (IOException e){
            logger.info(SERVER_SECRET_FILENAME+" file doesn't exists or something went wrong in reading that file. Generating a new secret for the server.");
            String newSecret = generateRandomString(32);
            try {
                FileOperation.writeFile(userDataPaths.getCurrentJarPath(), SERVER_SECRET_FILENAME, newSecret.getBytes());
            } catch (IOException ex) {
                ex.printStackTrace();
            }
            logger.info("Newly generated secret is now written to the filesystem for persistence.");
            return newSecret;
        }
    }
}
```

first this code mean that it will generate the token secret key by reading the file `server_secret.txt` but in exception when the file is not exsited in the server the key will be generated randomly by the number `1234` with 32 randomly(And we make sure there is no file name `server_secret.txt` in the server)

3.Role.java

``` java
@Table(name = "roles")
public class Role {
    @Id
    @Column
    private String name;

    @Column
    private Integer value; //higher the value, more the privilege. By this logic, admin is supposed to
    // have the highest value
}
```

this code say that the highernum for `userId` will have the higher priviledge, we can take advantage of this


### the solution

we already have all the information how we can modified our user account for getting the admin access

First, we go to the website `jwt.io`

![image](https://hackmd.io/_uploads/rJ10-s-PWl.png)


this is our jwt token for the user account

, next thing we will do is changing the `role` -> `Admin` , `email` -> `admin` and `userId` -> `2` , actually the ID you can put any numbers that is greater than the value 1 in default

With `secret_key` as I said before, it just the string `1234` for create the token randomly with 32 characters

![image](https://hackmd.io/_uploads/HJ0aMjWvWl.png)

just like this




## Final

Go back to the bookshelf website and inspect your web
Go to the `storage` index and open for the local storage
![image](https://hackmd.io/_uploads/Sk2LQiZvbe.png)

==> You will have to paste the payload and your modifed token into the `token-payload` and `auth-token`

Reload the page and there is your flag



**Flag**: `picoCTF{w34k_jwt_n0t_g00d_d7c2e335}`

**Author**:KhoaVo

**Solved**: February 5,2026