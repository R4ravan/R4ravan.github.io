---
title: "PortSwigger: Broken Brute-Force Protection, IP Block"
date: 2026-11-09 10:02:00 +0530
categories: [PortSwigger, Authentication]
tags: [portswigger, authentication, practitioner, burp-suite, brute-force]
image:
  path: /assets/img/posts/covers/portswigger-broken-brute-force-protection-ip-block.svg
  alt: "Broken Brute-Force Protection, IP Block"
description: "The login blocks your IP after 3 consecutive failed logins, but the failure counter is reset by any successful login. By interleaving valid wiener logins between carlos guesses, I kept the IP unblocked and brute-forced carlos's password."
---

**Platform:** PortSwigger Web Security Academy  
**Category:** Authentication (Brute-Force Protection Logic Flaw)  
**Difficulty:** Practitioner  
**Date:** 2026-11-09

---

## 🎯 Objective

This lab is vulnerable due to a logic flaw in its password brute-force protection. The goal is to brute-force the password for `carlos`, then log in and access his account page.

- **Your credentials:** `wiener:peter`
- **Victim's username:** `carlos`
- A [candidate password list](https://portswigger.net/web-security/authentication/auth-lab-passwords) is provided by the lab.

## 🔍 Recon & Analysis

First I logged in normally as `wiener:peter` to confirm the happy path. A valid login returns a `302` redirect to the account page:

```http
POST /login HTTP/2
Host: <lab-id>.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

username=wiener&password=peter
```

```http
HTTP/2 302 Found
Location: /my-account?id=wiener
```

![Valid wiener login lands on the account page](https://raw.githubusercontent.com/R4ravan/web-security-writeups/main/portswigger/authentication/images/kali-linux-2026.1-vmware-amd64-2026-11-08-22-29-45.png)

Next I probed `carlos` in Repeater. A single wrong guess returns `200` with **"Incorrect password"**:

![Single failure: Incorrect password](https://raw.githubusercontent.com/R4ravan/web-security-writeups/main/portswigger/authentication/images/kali-linux-2026.1-vmware-amd64-2026-11-08-22-33-46.png)

But after **3 incorrect logins in a row**, the server stops checking the password and returns a block message instead:

```
You have made too many incorrect login attempts.
Please try again in 1 minute(s).
```

![IP block after too many failures](https://raw.githubusercontent.com/R4ravan/web-security-writeups/main/portswigger/authentication/images/kali-linux-2026.1-vmware-amd64-2026-11-08-22-33-54.png)

**The logic flaw:** the IP block triggers after **3 consecutive failed logins**, but the failure counter is **reset by any successful login**. So if I slip a genuine `wiener:peter` login in before the limit is reached, the counter never hits the threshold and the block never fires. This lets me brute-force `carlos` indefinitely.

## 💥 Exploitation

The plan: a **Pitchfork** Intruder attack with payload positions in both `username` and `password`, interleaving a valid `wiener:peter` login between `carlos` guesses so consecutive failures never reach 3.

> PortSwigger's model solution resets before **every** guess (`wiener, carlos, wiener, carlos, …`). I reset after **every two** `carlos` guesses (`wiener, carlos, carlos, …`) — since the threshold is 3 consecutive failures, a maximum of 2 in a row is still safe and uses fewer requests.
{: .prompt-tip }

**1. Send the request to Intruder and set payload positions.**

Enter an invalid username/password, send the `POST /login` request to Intruder, and set it to **Pitchfork** with positions on both `username` and `password`.

**2. Force single-threaded, ordered requests.**

This is critical. In **Resource Pool**, create a custom pool with **Maximum concurrent requests = 1**. Sending one request at a time guarantees the login attempts reach the server in the correct order — otherwise parallel requests could fire 3+ `carlos` failures before a `wiener` reset lands and re-trigger the block.

![Resource pool set to 1 concurrent request](https://raw.githubusercontent.com/R4ravan/web-security-writeups/main/portswigger/authentication/images/kali-linux-2026.1-vmware-amd64-2026-11-08-22-59-07.png)

**3. Build the aligned payload lists.**

Position 1 (username) alternates so a reset lands on schedule — `wiener` first, then `carlos` repeated through the list:

```
wiener
carlos
carlos
wiener
carlos
carlos
...
```

![Payload set 1: username list](https://raw.githubusercontent.com/R4ravan/web-security-writeups/main/portswigger/authentication/images/kali-linux-2026.1-vmware-amd64-2026-11-08-22-49-54.png)

Position 2 (password) must line up with it: a valid `peter` wherever the username is `wiener`, and the candidate passwords wherever it's `carlos`. I used `awk` to inject `peter` after every two candidates:

```bash
awk '{print} NR%2==0{print "peter"}' pass.txt
```

Then I added a leading `peter` so the very first request (`wiener`) is a valid, counter-clearing login:

```
peter
123456
password
peter
12345678
qwerty
peter
...
```

![Building the interleaved password list with awk](https://raw.githubusercontent.com/R4ravan/web-security-writeups/main/portswigger/authentication/images/kali-linux-2026.1-vmware-amd64-2026-11-08-22-49-39.png)

So each group of three requests is: `wiener:peter` (success → counter reset), `carlos:guess1` (fail), `carlos:guess2` (fail), then reset again.

![Payload set 2 loaded + results](https://raw.githubusercontent.com/R4ravan/web-security-writeups/main/portswigger/authentication/images/kali-linux-2026.1-vmware-amd64-2026-11-08-22-55-38.png)

**4. Run the attack and read the status codes.**

Start the attack. When it finishes, filter out `200` responses and sort by username. Every `wiener:peter` row returns `302` (expected resets). Every wrong `carlos` guess returns `200`. The single `carlos` row that returns **`302`** (length `188`, matching the valid-login redirect) is the hit — visible highlighted in the results above:

```
Request  Payload1  Payload2   Status  Length
...
65       carlos    batman     302     188   ← cracked
```

So `carlos`'s password is **`batman`**.

**5. Confirm in Repeater.**

```http
POST /login HTTP/2
...
username=carlos&password=batman
```

```http
HTTP/2 302 Found
Location: /my-account?id=carlos
```

![Repeater confirms carlos:batman → 302](https://raw.githubusercontent.com/R4ravan/web-security-writeups/main/portswigger/authentication/images/kali-linux-2026.1-vmware-amd64-2026-11-08-22-59-17.png)

**6. Log in via the browser** as `carlos:batman` and open the account page. Lab solved.

![Congratulations, you solved the lab](https://raw.githubusercontent.com/R4ravan/web-security-writeups/main/portswigger/authentication/images/kali-linux-2026.1-vmware-amd64-2026-11-08-23-00-18.png)

## 🧠 Root Cause

- **Counter reset on success:** the brute-force defence counts *consecutive* failures and clears the count on any successful login from the **same IP**, regardless of which account succeeded.
- **Shared IP-based counter across accounts:** a successful login to *my* account (`wiener`) cleared the failure count that should have protected *another* account (`carlos`).
- **No independent per-account lockout:** `carlos` had no limit of its own, so an attacker who controls any valid account can keep the IP "clean" indefinitely.

## 🛡️ Remediation

- Track failed attempts **per target account**, not only per IP, and never let a success on one account reset another account's counter.
- Use an **absolute / escalating lockout** (temporary lock + exponential back-off) that a successful login elsewhere cannot clear.
- Add **CAPTCHA** after a few failures and **rate-limit** login requests server-side.
- Enforce **strong password policies** and **multi-factor authentication** so guessable passwords like `batman` aren't enough on their own.
- Alert/monitor on high-volume login patterns from a single source, even when interleaved with successes.

## 📝 Key Takeaways

- "Brute-force protection exists" ≠ "brute-force is prevented" — always test *how* the counter is reset and *what scope* it applies to.
- A reset-on-success design is a classic logic flaw: feed it successes to keep the gate open.
- In Burp, **Pitchfork** + a carefully aligned payload list lets you interleave valid logins; set **max concurrent requests to 1** so the ordering (and therefore the reset timing) actually holds.
- Watch the **status code / response length**, not just the body — the lone `302` stood out instantly against a wall of `200`s.

## 🔗 References

- [PortSwigger: Lab — Broken brute-force protection, IP block](https://portswigger.net/web-security/authentication/password-based/lab-broken-bruteforce-protection-ip-block)
- [PortSwigger: Authentication vulnerabilities](https://portswigger.net/web-security/authentication)
- [OWASP: Blocking Brute Force Attacks](https://owasp.org/www-community/controls/Blocking_Brute_Force_Attacks)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

---

*Originally published in my [web-security-writeups](https://github.com/R4ravan/web-security-writeups/blob/main/portswigger/authentication/broken-brute-force-protection-ip-block.md) repository.*
