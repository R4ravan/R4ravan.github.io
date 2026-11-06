---
title: "TryHackMe: Linux Fundamentals Part 1"
date: 2026-11-03 10:01:00 +0530
categories: [TryHackMe, Linux]
tags: [tryhackme, linux, easy]
---

**Platform:** TryHackMe  
**Category:** Linux Basics  
**Difficulty:** Easy  
**Date:** 2026-11-03  
**Room:** [linuxfundamentalspart1](https://tryhackme.com/room/linuxfundamentalspart1)  

The first room in TryHackMe's Linux Fundamentals series. It introduces the Linux terminal and covers the basics: identifying the current user, printing output, navigating the filesystem, searching files, and combining commands with shell operators.

---

## Task 1 — Introduction

Linux is everywhere: web servers, car infotainment systems, point-of-sale tills, traffic-light controllers, phones and more. The room provides an in-browser Ubuntu machine to practise on.

**Action:** Start the machine and wait for the terminal to load. No questions to answer.

---

## Task 2 — Talking to Linux

| Command | Purpose |
|---------|---------|
| `whoami` | Shows which user you are logged in as |
| `echo`   | Prints the text you give it |

```bash
tryhackme@linux1:~$ whoami
tryhackme
tryhackme@linux1:~$ echo TryHackMe
TryHackMe
tryhackme@linux1:~$ echo "hello world"
hello world
```

> Multi-word strings should be wrapped in quotes. Use the ↑/↓ arrow keys to cycle through previous commands.

**Q1. What command would we use to find out who we are on the system?**
`whoami`

**Q2. What command would we use to output some text?**
`echo`

---

## Task 3 — Finding Your Way Around

| Command | Purpose |
|---------|---------|
| `ls`  | List the contents of the current folder |
| `cd`  | Change directory |
| `cat` | Display the contents of a file |
| `pwd` | Print the current working directory |

```bash
tryhackme@linux1:~$ ls
access.log  folder1  folder2  folder3  folder4
tryhackme@linux1:~$ ls folder*
folder1:
access.log  passwords.txt

folder2:

folder3:

folder4:
tryhackme@linux1:~$ cd folder1
tryhackme@linux1:~/folder1$ cat passwords.txt
password123
tryhackme@linux1:~/folder1$ pwd
/home/tryhackme/folder1
```

**Q1. Run `ls` in the current folder. How many folders are there?**
`4`

**Q2. One of those folders contains a file. Which folder is it?**
`folder1`

**Q3. Use `cat` to read the file within this folder. What does it say?**
`password123`

---

## Task 4 — Let the Machine Do the Searching

| Command | Purpose |
|---------|---------|
| `find` | Search for files by name — e.g. `find -name passwords.txt` |
| `grep` | Search for text inside files — e.g. `grep "password123" passwords.txt` |

The home folder contains a web-server log, `access.log`, that is hundreds of lines long. Instead of scrolling through it, let `grep` find the flag:

```bash
tryhackme@linux1:~/folder1$ cd /home/tryhackme
tryhackme@linux1:~$ grep "THM" access.log
... THM{ACCESS} ...
```

**Q1. After using `grep "THM" access.log`, a flag was found. What is the flag?**
`THM{ACCESS}`

---

## Task 5 — Shell Operators (Combining Commands)

| Operator | Behaviour |
|----------|-----------|
| `&`  | Runs a command in the background so you can keep working |
| `&&` | Runs the second command only after the first one finishes |
| `>`  | Redirects output to a file, **overwriting** it |
| `>>` | Redirects output to a file, **appending** to the end |

```bash
# Run two commands in sequence
tryhackme@linux1:~$ cd folder1 && ls

# Overwrite a file
tryhackme@linux1:~$ echo hello > welcome.txt

# Append to a file
tryhackme@linux1:~$ echo world >> welcome.txt
tryhackme@linux1:~$ cat welcome.txt
hello
world
```

**Q1. What operator waits for the first command to finish before running the next command?**
`&&`

**Q2. What redirector allows us to save the output of a command without overwriting the contents of a file?**
`>>`

---

## Key Takeaways

- `whoami` tells you what level of access you have — important whenever you switch users.
- `ls`, `cd`, `cat` and `pwd` cover almost all basic navigation.
- `grep` saves huge amounts of time when digging through logs. `grep -R` searches recursively across directories.
- `>` overwrites, `>>` appends — useful for saving command output and building repeatable workflows.

**Next room:** [Linux Fundamentals Part 2](https://tryhackme.com/room/linuxfundamentalspart2)

---

*Originally published in my [web-security-writeups](https://github.com/R4ravan/web-security-writeups/blob/main/tryhackme/linux-fundamentals/01-linux-fundamentals-part-1.md) repository.*
