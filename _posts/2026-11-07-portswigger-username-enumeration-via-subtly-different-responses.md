---
title: "PortSwigger: Username Enumeration via Subtly Different Responses"
date: 2026-11-07 06:30:00 +0530
categories: [PortSwigger, Authentication]
tags: [portswigger, authentication, practitioner, burp-suite]
image:
  path: /assets/img/posts/covers/portswigger-username-enumeration-via-subtly-different-responses.svg
  alt: "Username Enumeration via Subtly Different Responses"
description: "The login form looks like it returns an identical error for every failed attempt — but one valid username triggers a response with a tiny, almost invisible difference. Here's how to catch it with Burp's Grep - Extract."
---

**Platform:** PortSwigger Web Security Academy  
**Category:** Authentication → Username Enumeration + Password Brute-Force  
**Difficulty:** Practitioner  
**Date:** 2026-11-07  
**Severity:** High — the login response discloses whether a username is valid through a tiny inconsistency in the error message, and with no rate limiting an attacker can enumerate a valid account and then brute-force its password, leading to full account takeover.  

---

## 🎯 Objective

This lab is the sneaky cousin of the classic *username enumeration via different responses* challenge. Here the application **tries** to return one generic message — `Invalid username or password.` — for every failed login, no matter which field was wrong. On the surface, enumeration should be impossible.

But the developers slipped. For one valid username, the error message is **subtly different** from all the others. The difference is so small you will never spot it by eye. The goal is to:

1. Enumerate the valid username by catching that tiny inconsistency.
2. Brute-force that user's password.
3. Log in and reach the account page to solve the lab.

PortSwigger supplies two wordlists:

- **Candidate usernames** (101 entries)
- **Candidate passwords** (100 entries)

---

## 🔍 Recon & Analysis

With Burp proxying traffic, I submitted a junk login — `username=test`, `password=test` — and read the response:

```
Invalid username or password.
```

So far this looks secure. Unlike the *different responses* lab, there is no obvious `Invalid username` vs `Incorrect password` split — every failure returns the same generic sentence, and the responses are almost exactly the same length (around 3463–3464 bytes). The status code is `200` for every failed attempt too.

That means **none of the usual signals work here**:

- Status code → always `200`
- Response length → essentially identical across all usernames
- Error message text → looks the same every time

When the obvious oracles are all flat, the difference is hiding *inside* the response body in a way a human won't notice. This is exactly what Burp's **Grep - Extract** feature is built for: pull out the exact error string from every response and compare them character-for-character.

I located the `POST /login` request in **Proxy → HTTP history** and sent it to **Intruder**:

```
POST /login HTTP/2
Host: <lab-id>.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
...
username=test&password=test
```

---

## 💥 Exploitation

### 1. Enumerate the valid username

In Intruder I chose a **Sniper** attack, cleared the auto-selected positions, and marked **only the username** as the payload position, leaving the password as a throwaway value:

```
username=§test§&password=test
```

I loaded the **candidate usernames** list (101 payloads) as a Simple list.

The important step is the **Grep - Extract** rule. Under **Settings → Grep - Extract**, I clicked **Add**, scrolled through the sample response until I found the `Invalid username or password.` message, and highlighted the full text of it. Burp then extracts that exact string from every response and drops it into its own results column.

Then I started the attack.

> **Key observation — every obvious column is flat.** All 101 requests returned **HTTP 200**, and the lengths barely moved (3463 vs 3464). If you only look at status or length, nothing stands out. The whole attack lives in the extracted-error column.

When the attack finished, I sorted (or filtered) on the extracted error column. The fastest way to find the odd one is a **negative filter**: in the results filter I searched for the literal string `Invalid username or password.` with **Negative search** enabled — that is, *show me every row that does NOT contain this exact message*.

Only **one** request survived the filter:

```
Request 69   →   payload: apache   →   status 200   →   length 3463
```

Every other username returned the message exactly as `Invalid username or password.` — but `apache` returned something that *didn't match*. Looking closely at that one response reveals the flaw:

- All invalid usernames → `Invalid username or password.` (ends with a full stop)
- `apache` → `Invalid username or password` followed by a **trailing space**, with **no full stop**

That single whitespace-vs-period difference is the entire vulnerability. It is invisible on the rendered page and invisible to the length column at a glance, but it uniquely marks the valid account.

**Valid username: `apache`**

### 2. Brute-force the password

Back in the **Positions** tab I clicked **Clear §**, hard-coded the username I had found, and moved the payload position onto the **password**:

```
username=apache&password=§invalid-password§
```

I cleared the username payloads, loaded the **candidate passwords** list (100 payloads), and started a second attack.

> **Key observation — now the status code is the signal.** A failed login returns `200`; a successful one redirects. So this time I sorted by the **Status** column.

Exactly one request broke the pattern with a **`302 Found`**:

```
Request 35   →   payload: jordan   →   status 302   →   length 188
```

The response confirmed the login and issued a fresh authenticated session:

```
HTTP/2 302 Found
Location: /my-account?id=apache
Set-Cookie: session=<NEW-AUTHENTICATED-SESSION>; Secure; HttpOnly; SameSite=None
```

**Valid password: `jordan`**

### 3. Log in and solve

Using the recovered credentials **`apache : jordan`**, I authenticated and loaded the account page:

```
Your username is: apache
Your email is: apache@normal-user.net
```

The lab flipped to **Solved**. ✅

**Credentials recovered:** `apache : jordan`

---

## 🧠 Root Cause

- **An inconsistent "generic" error message.** The developers intended a single uniform failure message, which is the correct defence. But one code path produced `Invalid username or password` with a trailing space and no period instead of the canonical `Invalid username or password.`. That tiny inconsistency re-introduces the exact username oracle the generic message was supposed to remove.
- **Machine-readable differences a human can't see.** Enumeration does not need an obvious message like `Incorrect password`. *Any* measurable, reproducible difference — a stray space, a byte of length, a few milliseconds of timing — is enough. Burp's Grep - Extract turns an invisible one-character difference into a sortable column.
- **No rate limiting or account lockout.** The server happily processed 101 username guesses and 100 password guesses back-to-back with no throttling, CAPTCHA, or lockout, so the leak is directly and quickly exploitable.

---

## 🛡️ Remediation

- **Return one byte-for-byte identical failure response** for every unsuccessful login — same message, same punctuation, same whitespace, same status code, same length. Centralise the message in a single constant so no code path can drift.
- **Normalise processing time** so valid-username and invalid-username paths take the same amount of work (validate the password even when the username does not exist).
- **Enforce rate limiting and account lockout** per account and per source IP, with CAPTCHA after a few failures.
- **Add multi-factor authentication** so a guessed password alone is not enough to take over an account.

---

## 📝 Key Takeaways

- **"Generic" isn't generic unless it's identical every time.** A single stray space or missing period is a full username oracle — the defence only works if the response is literally the same for every failure.
- **When status and length are flat, use Grep - Extract.** Extracting the exact error string and comparing it across all requests surfaces differences the eye and the length column miss. A negative-match filter (`show rows that don't contain the normal message`) finds the odd one instantly.
- **Pick the right signal for each phase.** Username enumeration lived in the extracted-message column; password brute-force surfaced in the status code (`302` vs `200`). Always compare more than one column.
- **Enumeration + no rate limiting = account takeover.** Each weakness is minor alone; chained together they give a full credential compromise.

---

## 🔗 References

- [PortSwigger — Username enumeration via subtly different responses](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-subtly-different-responses)
- [PortSwigger — Vulnerabilities in password-based login](https://portswigger.net/web-security/authentication/password-based)
- [OWASP — Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP WSTG — Testing for Account Enumeration](https://owasp.org/www-project-web-security-testing-guide/)
