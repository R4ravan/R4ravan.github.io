---
title: "PortSwigger: Username Enumeration via Account Lock"
date: 2026-11-09 23:10:00 +0530
categories: [PortSwigger, Authentication]
tags: [portswigger, authentication, practitioner, burp-suite, username-enumeration, brute-force]
image:
  path: /assets/img/posts/covers/portswigger_account_lock_cover_with_title.png
  alt: "Username Enumeration via Account Lock"
description: "The login form locks an account after too many failed attempts — and that lock message leaks which usernames are valid. I enumerated the username 'announce' through the lock response, then brute-forced its password despite the lock because the correct password still produced a distinct, shorter response."
---

**Platform:** PortSwigger Web Security Academy  
**Category:** Authentication (Username Enumeration via Account-Lock Logic Flaw)  
**Difficulty:** Practitioner  
**Date:** 2026-11-09

---

## 🎯 Objective

This lab is vulnerable to username enumeration. It uses account locking, but the locking logic contains a flaw. To solve the lab I had to:

1. **Enumerate a valid username**,
2. **Brute-force that user's password**, then
3. **Log in and access the account page**.

- A [candidate username list](https://portswigger.net/web-security/authentication/auth-lab-usernames) and [candidate password list](https://portswigger.net/web-security/authentication/auth-lab-passwords) are provided by the lab.

## 🔍 Recon & Analysis

The login form submits a standard URL-encoded `POST /login`:

```http
POST /login HTTP/2
Host: <lab-id>.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=test&password=example
```

A single wrong username/password returns `200` with a generic **"Invalid username or password."** — no distinction between a bad username and a bad password. So one request tells me nothing.

**The observation that breaks it:** when the *same* username is submitted several times with wrong passwords, a **valid** account eventually gets locked and the server switches to a different message:
