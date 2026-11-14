---
title: "PortSwigger: 2FA simple bypass"
date: 2026-11-14 15:22:00 +0530
categories: [PortSwigger, Authentication]
tags: [authentication, 2fa, mfa-bypass, broken-access-control, apprentice, burp-suite]
image:
  path: /assets/img/posts/covers/2fa-simple-bypass.png
  alt: "2FA simple bypass"
description: "Bypassing a broken two-factor authentication flow by skipping the verification step and navigating straight to the account page after login."
---

**Platform:** PortSwigger Web Security Academy
**Category:** Authentication
**Difficulty:** Apprentice
**Date:** 2026-11-14 15:22 IST

## 🎯 Objective

This lab's two-factor authentication can be bypassed. I already have a valid username and password for the victim, but no access to their 2FA verification code. The goal is to access Carlos's account page.

- My credentials: `wiener:peter`
- Victim's credentials: `carlos:montoya`

## 🔍 Recon & Analysis

First I logged in with my own account to understand the authentication flow.

![Login page with wiener credentials](/assets/img/posts/2fa-simple-bypass/01-login-wiener.png)

After submitting valid credentials, the app redirected me to a second step asking for a 4-digit security code.

![2FA prompt asking for a 4-digit security code](/assets/img/posts/2fa-simple-bypass/02-2fa-prompt.png)

The verification code is sent by email. I opened the Email client and confirmed the code was delivered there.

![Email client showing the security code](/assets/img/posts/2fa-simple-bypass/03-email-security-code.png)

I completed my own 2FA and reached the account page. I noted the URL used after login: `/my-account`.

![My account page for wiener with the my-account URL noted](/assets/img/posts/2fa-simple-bypass/04-my-account-wiener.png)

The key question was whether the account page actually enforces that 2FA was completed, or whether it only sits "after" the 2FA step by convention. I inspected the requests in Burp to compare the 2FA POST (`/login2`) against the `/my-account` request.

![Burp Comparer showing the login2 and my-account requests side by side](/assets/img/posts/2fa-simple-bypass/05-comparer-requests.png)

The session cookie is issued at the password step, before the code is ever verified. That strongly suggested the `/my-account` page is reachable as soon as the password stage succeeds.

## 💥 Exploitation

1. Log out of my own account to clear state.

2. Log in using the victim's credentials.

   ```http
   POST /login HTTP/2
   Host: 0a4300f6044c237981fc85f100ea0009.web-security-academy.net
   Content-Type: application/x-www-form-urlencoded

   username=carlos&password=montoya
   ```

   ![Login page with carlos credentials](/assets/img/posts/2fa-simple-bypass/06-login-carlos.png)

3. The app then prompts for the 4-digit code at `/login2`. Instead of entering a code I don't have, I manually changed the URL in the browser to navigate straight to `/my-account`.

   ![Manually editing the browser URL to /my-account](/assets/img/posts/2fa-simple-bypass/07-url-change-my-account.png)

   ```http
   GET /my-account HTTP/2
   Host: 0a4300f6044c237981fc85f100ea0009.web-security-academy.net
   ```

4. The account page loaded for Carlos without ever supplying the 2FA code, and the lab was marked solved.

   ![Carlos account page loaded and lab marked as solved](/assets/img/posts/2fa-simple-bypass/08-solved-carlos-account.png)

## 🧠 Root Cause

The 2FA step is not enforced server-side on protected pages. An authenticated session is granted after the password stage (step 1), and the verification code step (`/login2`) is treated as a separate, optional screen rather than a gate. Because `/my-account` only checks that a session exists — not that 2FA was completed — I could skip the code entirely by requesting the page directly.

## 🛡️ Mitigation

- Treat 2FA as a mandatory state in the session: do not grant a fully authenticated session until the verification code is validated.
- Mark the session as "password-verified, pending 2FA" after step 1, and reject requests to protected resources until the 2FA state is set to "verified".
- Enforce this server-side on every protected endpoint, not just by redirect order in the UI.
- Invalidate the pending session if 2FA is abandoned or fails repeatedly.

## 📝 Key Takeaways

- A multi-step login is only as strong as its weakest server-side check; UI order is not access control.
- If a session cookie is issued before the second factor is verified, the second factor can often be skipped.
- Always test protected pages by requesting them directly, out of the intended flow.
- Broken 2FA frequently reduces to a plain broken-access-control issue.

## 🔧 Tools Used

- Burp Suite (Repeater, Comparer)
- Firefox with PwnFox
- Email client (PortSwigger lab)
