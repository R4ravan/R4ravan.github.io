---
title: "OverTheWire: Bandit — Level 0 to Level 11"
date: 2026-11-14 23:15:00 +0530
categories: [OverTheWire, Linux Fundamentals]
tags: [linux, ssh, wargame, bandit, overthewire, file-permissions, bash, text-processing]
image:
  path: /assets/img/posts/covers/overthewire-bandit-level-0-to-11.png
  alt: "OverTheWire Bandit Level 0 to 11"
description: "Walkthrough of OverTheWire Bandit levels 0 through 11, covering SSH login, tricky filenames, hidden files, file properties, text processing and basic encoding/ciphers."
---

**Platform:** OverTheWire (Bandit wargame)
**Category:** Linux Fundamentals / SSH
**Difficulty:** Beginner
**Date:** 2026-11-14 23:15 IST

## 🎯 Objective

Bandit is a beginner-friendly wargame hosted by OverTheWire. Each level gives you the credentials for a `banditN` user over SSH, and a goal to find the password for `banditN+1`. This post covers my solutions for Level 0 through Level 11, where I practiced basic Linux navigation, handling tricky filenames, finding files by properties, and simple text processing and decoding.

**Connection info used throughout:**
```bash
ssh banditN@bandit.labs.overthewire.org -p 2220
```

## 🔍 Recon & Analysis

Every level follows the same pattern: log in with the current level's password, read the goal, find the next password using basic shell commands (`ls`, `cat`, `find`, `grep`, `sort`, `file`, `strings`, `base64`, `tr`), then SSH into the next level with that password. The difficulty ramps up through filename edge cases, hidden files, file attributes, and simple encodings.

## 💥 Exploitation

### Level 0 → Level 1

**Goal:** Log in via SSH with username `bandit0` and password `bandit0`, then find the password for Level 1 in a file called `readme` in the home directory.

```bash
ssh bandit0@bandit.labs.overthewire.org -p 2220
```

```bash
bandit0@bandit:~$ ls
readme
bandit0@bandit:~$ cat readme
Congratulations on your first steps into the bandit game!!
Please make sure you have read the rules at https://overthewire.org/rules/
If you are following a course, workshop, walkthrough or other educational activity,
please inform the instructor about the rules as well and encourage them to
contribute to the OverTheWire community so we can keep these games free!

The password you are looking for is: 6y2kwnwK6grgvwvpvLaa2T1cpFEKOhNR
```

Simple `ls` + `cat` on a normal file. Password found directly in plain text.

### Level 1 → Level 2

**Goal:** The password is stored in a file literally named `-` in the home directory.

```bash
bandit1@bandit:~$ ls
-
bandit1@bandit:~$ cat ./-
PK8fYLZg2hnHSz83plBL1iEPKdD3QToB
```

A bare `-` filename is problematic because most commands treat a leading `-` as the start of a flag. Prefixing the filename with `./` forces the shell to treat it as a relative path instead of an option.

### Level 2 → Level 3

**Goal:** The password is in a file named `--spaces in this filename--`.

```bash
bandit2@bandit:~$ ls
--spaces in this filename--
bandit2@bandit:~$ cat '--spaces in this filename--'
error: unexpected argument '--spaces in this filename--' found

  tip: to pass '--spaces in this filename--' as a value, use '-- --spaces in this filename--'
```

Quoting alone wasn't enough here because the filename starts with `--`, which `cat` still parsed as an option indicator even inside quotes.

```bash
bandit2@bandit:~$ cat ./'--spaces in this filename--'
7ZZ2LFrykP2zEyvBl4m3clcL7tGYJPME
```

Adding `./` in front of the quoted name fixed it — same root cause as Level 1, combined with spaces needing quoting.

### Level 3 → Level 4

**Goal:** The password is in a hidden file inside the `inhere` directory.

```bash
bandit3@bandit:~$ ls
inhere
bandit3@bandit:~$ ls -a inhere
.  ..  ...Hiding-From-You
bandit3@bandit:~$ cat inhere/...Hiding-From-You
xzTXq1rDJQVVAzdv5cHq1TQytTWufAMq
```

Plain `ls` doesn't show dotfiles. The hidden file here is named `...Hiding-From-You` — a dot-prefixed name that only shows up with `-a`.

### Level 4 → Level 5

**Goal:** The password is in the only human-readable file among ten files in `inhere`.

```bash
bandit4@bandit:~$ ls inhere/
-file00  -file01  -file02  -file03  -file04  -file05  -file06  -file07  -file08  -file09
bandit4@bandit:~$ file ./inhere/*
./inhere/-file00: data
./inhere/-file01: data
./inhere/-file02: Non-ISO extended-ASCII text, with NEL line terminators, with escape sequences
./inhere/-file03: data
./inhere/-file04: data
./inhere/-file05: data
./inhere/-file06: data
./inhere/-file07: ASCII text
./inhere/-file08: data
./inhere/-file09: Non-ISO extended-ASCII text, with no line terminators
bandit4@bandit:~$ cat ./inhere/-file07
6C7h9GD8M6ai5nr7wo1RonrzFjj9yIrG
```

`file` identifies file type by content, not extension. Only `-file07` was reported as plain `ASCII text`, so that was the right one to read.

### Level 5 → Level 6

**Goal:** Find a file under `inhere` that is human-readable, exactly 1033 bytes, and not executable.

```bash
bandit5@bandit:~$ find ./inhere -size 1033c
./inhere/maybehere07/.file2
bandit5@bandit:~$ cat ./inhere/maybehere07/.file2
pXa26xhMWaC2SvDotA4r9EgZkulOeSBW
```

`find` with `-size 1033c` matched the exact byte count directly, skipping the need to check permissions and readability manually since only one file matched the size.

### Level 6 → Level 7

**Goal:** The password is stored somewhere on the whole server, owned by user `bandit7`, group `bandit6`, exactly 33 bytes in size.

```bash
bandit6@bandit:~$ find / -size 33c -user bandit7 -group bandit6 2>/dev/null
/var/lib/dpkg/info/bandit7.password
bandit6@bandit:~$ cat /var/lib/dpkg/info/bandit7.password
Bmnnvf82KzQlfxgAI2d1zYbr1u9pr3E3
```

Here the search scope widened to the entire filesystem (`/`). Combining `-user`, `-group`, and `-size` narrowed it to one exact match, and `2>/dev/null` suppressed the permission-denied noise from directories I couldn't read.

### Level 7 → Level 8

**Goal:** The password in `data.txt` is next to the word `millionth`.

```bash
bandit7@bandit:~$ wc data.txt
  98567  197133 4184396 data.txt
bandit7@bandit:~$ grep millionth data.txt
millionth       VR1ljMayciFxbnUokuQmJFw6QC9VKtub
```

With a ~98,000-line file, `grep` for the known keyword was far faster than manually scrolling through it.

### Level 8 → Level 9

**Goal:** The password is the only line in `data.txt` that occurs exactly once.

```bash
bandit8@bandit:~$ sort data.txt | uniq -u
EjmOSvuAu7sGAHqHVcBDPirRe9T03kxl
```

`uniq -u` only prints lines with no duplicates, but it requires sorted input first — hence piping through `sort` before it.

### Level 9 → Level 10

**Goal:** The password is one of the few human-readable strings in `data.txt`, preceded by several `=` characters.

```bash
bandit9@bandit:~$ file data.txt
data.txt: data
bandit9@bandit:~$ strings data.txt | grep ========
========== the
========== password
========== is
========== B0s2khmbT9u0geKuOoVGW3JZKhndE3BG
```

The file itself was binary (`data`), so `cat` would've shown garbage. `strings` extracts printable text from binary data, and grepping for the `=` pattern isolated the relevant lines.

### Level 10 → Level 11

**Goal:** `data.txt` contains base64-encoded data.

```bash
bandit10@bandit:~$ file data.txt
data.txt: ASCII text
bandit10@bandit:~$ cat data.txt
VGhlIHBhc3N3b3JkIGlzIHBZZk9ZNkh3VXNEajVyTDlVdnloVTdNQ212OHZONVJvCg==
bandit10@bandit:~$ base64 -d data.txt
The password is pYfOY6HwUsDj5rL9UvyhU7MCmv8vN5Ro
```

The `==` padding at the end and the characteristic alphabet were the giveaway for base64. `base64 -d` decoded it directly to plaintext.

### Level 11 → Level 12

**Goal:** `data.txt` has all letters rotated by 13 positions (ROT13).

```bash
bandit11@bandit:~$ cat data.txt
Gur cnffjbeq vf TEBbmJCB8DlA0zTewHxVQ0JPLxMvDkeA
bandit11@bandit:~$ cat data.txt | tr a-zA-Z n-za-mN-ZA-M
The password is GROozWPO8QyN0mGrjUkID0WCYkZiQxrN
```

`tr a-zA-Z n-za-mN-ZA-M` maps each letter to the one 13 positions ahead, wrapping around — a one-liner ROT13 decode without needing a dedicated tool.

## 🧠 Root Cause / Core Concepts

None of these levels involve a "vulnerability" in the usual sense — Bandit is designed to build fluency with:

- Shell argument parsing quirks (`-`, `--`, leading dashes in filenames)
- Hidden files and `ls -a`
- File type identification independent of extension (`file`, `strings`)
- Targeted search by file size, owner, and group (`find`)
- Text processing and deduplication (`grep`, `sort`, `uniq`, `wc`)
- Basic encodings and classical ciphers (`base64`, ROT13 via `tr`)

## 🛡️ Mitigation

Not applicable in the traditional sense since this is an educational wargame, not a live system being secured. The real "defense" lesson is the inverse: these are exactly the techniques an attacker or auditor uses to locate secrets left in flat files, binaries, or misnamed files on a real system — so in production, sensitive values like passwords should never sit in plaintext files, even "hidden" ones, since none of the obfuscation shown here (odd filenames, encoding, hidden paths) stopped discovery once the right command was used.

## 📝 Key Takeaways

- `./` before a filename neutralizes leading dashes that the shell would otherwise treat as flags.
- `ls -a` and `find` are essential for files hiding in plain sight.
- `file` and `strings` tell you what you're actually dealing with before you try to read it.
- `grep`, `sort | uniq -u`, `wc`, `find -size/-user/-group` turn "search by hand" into a one-liner.
- Base64 and ROT13 are trivially reversible — never real security, just encoding/obfuscation.

## 🔧 Tools Used

- OpenSSH client
- Core GNU utilities: `ls`, `cat`, `find`, `file`, `strings`, `grep`, `sort`, `uniq`, `wc`, `tr`, `base64`
