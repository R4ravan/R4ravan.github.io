---
title: "PortSwigger: Username Enumeration via Account Lock"
date: 2026-11-09 23:10:00 +0530
categories: [PortSwigger, Authentication]
tags: [portswigger, authentication, practitioner, burp-suite, username-enumeration, brute-force]
image:
  path: /assets/img/posts/covers/account_lock_cover_banner.png
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

```
You have made too many incorrect login attempts.
Please try again in 1 minute(s).
```

An **invalid** username never triggers this — the server has no account to lock, so it keeps returning the generic error. That difference is the oracle: **only real usernames can be locked.** A valid account's responses become longer (they carry the lock message), and that length difference is trivially visible in Intruder.

## 💥 Exploitation

### Phase 1 — Enumerate the username (Cluster Bomb + Null payloads)

Send `POST /login` to Intruder and choose **Cluster bomb**. Put a payload position on `username`, and add a **second, empty** position at the end of the body so the same username gets replayed multiple times:

```
username=§invalid-username§&password=example§§
```

- **Payload set 1 (username):** the candidate username list.
- **Payload set 2 (the empty position):** **Null payloads → generate 5**. This replays every username 5 times in a row, enough to trip the lock on whichever one is real.

> No need to paste each username five times by hand — **Null payloads** does the repetition for you, and Cluster bomb pairs every username with all 5 nulls automatically.
{: .prompt-tip }

Run it, then sort by **Length**. Every username sits at `3344`, except **`announce`**, whose three later hits jump to `3396` — those are the responses carrying the account-lock message. The render pane confirms it: *"You have made too many incorrect login attempts."*

![Cluster bomb: `announce` responses are longer (3396) — account locked, username is valid](/assets/img/posts/images/username-enumeration-via-account-lock/kali-linux-2026.1-vmware-amd64-2026-11-09-22-28-31.png)

**Valid username: `announce`.**

### Phase 2 — Brute-force the password (Sniper + grep)

Create a fresh Intruder attack on the same request, this time **Sniper**. Hard-code the username to `announce` and put the single payload position on `password`:

```
username=announce&password=§example§
```

- **Payload set:** the candidate password list.
- Add a **Grep – Extract** rule for the error-message text so each row shows whether an error was returned.

Here's the logic flaw that makes this work: even while the account is "locked", the server **still evaluates the submitted password**. Wrong guesses render the lock message (longest responses, `3396`); the **correct** password renders a *different* page with **no error block at all** — which shows up as a distinctly **shorter** response. Row 19, payload **`666666`**, drops to `3266` while everything around it sits at `3396`/`3344`:

![Sniper password brute-force: `666666` is the odd one out at 3266 (no error message)](/assets/img/posts/images/username-enumeration-via-account-lock/kali-linux-2026.1-vmware-amd64-2026-11-09-22-32-24.png)

**Password: `666666`.**

### Phase 3 — Confirm & log in

Confirm the pair in Repeater — a clean `200 OK` with no error message in the body:

```http
POST /login HTTP/2
...
username=announce&password=666666
```

```http
HTTP/2 200 OK
...
```

![Repeater confirms `announce:666666` returns a clean 200 with no error](/assets/img/posts/images/username-enumeration-via-account-lock/kali-linux-2026.1-vmware-amd64-2026-11-09-22-32-45.png)

> The lock resets after **1 minute**. Wait it out before logging in through the browser, otherwise the lock will still bounce you even with the correct credentials.
{: .prompt-tip }

Log in as `announce:666666` in the browser and open the account page. Lab solved.

![Congratulations, you solved the lab — My Account page for `announce`](/assets/img/posts/images/username-enumeration-via-account-lock/kali-linux-2026.1-vmware-amd64-2026-11-09-22-41-23.png)

## 🧠 Root Cause

- **Lock message is a username oracle:** the account-lock response is only ever returned for a *real* account, so an attacker learns which usernames exist purely from the error text / response length.
- **Lock doesn't actually gate password evaluation:** while "locked", the server still checks the password and renders a different page for a correct guess — leaking the valid password despite the supposed protection.
- **Short, fixed lockout:** a 1-minute reset gives an attacker effectively unlimited retries over time.

## 🛡️ Remediation

- Return a **single generic message** for every failed login — same text, same length — whether the username is unknown, the password is wrong, or the account is locked.
- Never leak lock state in a way that distinguishes valid from invalid usernames; keep the response **indistinguishable**.
- Use an **escalating lockout / back-off** that a correct password cannot bypass, and genuinely refuse evaluation while locked.
- Add **CAPTCHA** and server-side **rate limiting** after a few failures.
- Enforce strong passwords and **MFA**, so a guessable password like `666666` isn't enough on its own.

## 📝 Key Takeaways

- **Account locking can *create* an enumeration vuln** — a defence that behaves differently for real vs fake usernames is itself the leak.
- Here everything is `200`; the tell is **response length** and a **grep-extracted error message**, not the status code. Always diff responses, not just codes.
- **Cluster bomb + Null payloads** is the clean way to replay each username N times for enumeration; **Sniper + Grep Extract** isolates the one password whose response lacks the error block.
- Mind the **lock timer** — crack the password offline/in Intruder, then wait for the reset before the real login.

## 🔗 References

- [PortSwigger: Lab — Username enumeration via account lock](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-account-lock)
- [PortSwigger: Authentication vulnerabilities](https://portswigger.net/web-security/authentication)
- [OWASP: Blocking Brute Force Attacks](https://owasp.org/www-community/controls/Blocking_Brute_Force_Attacks)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

---

*Originally published in my [web-security-writeups](https://github.com/R4ravan/web-security-writeups/blob/main/portswigger/authentication/username-enumeration-via-account-lock.md) repository.*
