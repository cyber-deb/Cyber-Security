# 🥒 Pickle Rick — TryHackMe

> My first Capture The Flag (CTF) challenge.

**Platform:** [TryHackMe](https://tryhackme.com/)  
**Room:** Pickle Rick  
**Difficulty:** Easy  
**Category:** Web / Linux / Privilege Escalation  
**Status:** Completed ✅

---

## 📌 Overview

Pickle Rick is a beginner-friendly TryHackMe CTF focused on compromising a vulnerable web application and finding three ingredients belonging to Rick.

This was my **first CTF**, and it gave me practical experience with the basic penetration-testing workflow:

- Reconnaissance
- Port and service enumeration
- Web enumeration
- Source-code inspection
- Credential discovery
- Web authentication
- Command execution
- Linux filesystem enumeration
- Linux permissions
- Privilege escalation using `sudo`

---

## 🎯 Objective

Find all three ingredients hidden throughout the target machine.

The general attack path was:

```text
Reconnaissance
      ↓
Web Enumeration
      ↓
Credential Discovery
      ↓
Web Login
      ↓
Command Execution
      ↓
www-data
      ↓
Filesystem Enumeration
      ↓
Ingredient #1
      ↓
Ingredient #2
      ↓
Privilege Escalation
      ↓
Root
      ↓
Ingredient #3
```

---

# 🔎 1. Reconnaissance

I started by scanning the target machine using Nmap to identify open ports and running services.

```bash
nmap -sC -sV TARGET_IP
```

The scan revealed an HTTP service, so I focused on the web application.

### Key takeaway

Before attempting exploitation, it is important to understand the target's attack surface:

- What ports are open?
- What services are running?
- What versions are being used?
- What services are worth investigating?

---

# 🌐 2. Web Enumeration

I opened the HTTP service in a browser and inspected the website.

I also inspected the HTML source code for hidden information and comments.

During this process, I discovered a suspicious Base64-encoded string.

I decoded it multiple times using:

```bash
echo 'BASE64_STRING' | base64 -d
```

The decoded information eventually led to a clue named 'Rabbit Hole'.

I also discovered the username:

```text
R1ckRul3s
```

---

# 🔐 3. Credential Discovery

Further enumeration of the web application revealed the password:

```text
Wubbalubbadubdub
```

The credentials I discovered were therefore:

| Credential | Value |
|---|---|
| Username | `R1ckRul3s` |
| Password | `Wubbalubbadubdub` |

These credentials allowed access to the Rick Portal.

> ⚠️ These credentials are specific to this intentionally vulnerable TryHackMe machine.

---

# 🚪 4. Rick Portal

After logging into the portal, I found several sections, including a **Commands** page.

The page displayed a message indicating that command execution was disabled.

However, when I inspected the HTML source, I found that the command form was still present.

The relevant form contained an input named:

```text
command
```

and a submit button.

I tested whether commands were actually being executed.

I entered:

```bash
id
```

The server returned:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirmed that I had achieved **command execution as `www-data`**.

---

# 💻 5. Initial Access

The command execution gave me a low-privileged foothold on the target.

Current user:

```text
www-data
```

I checked the current working directory:

```bash
pwd
```

Then enumerated the files:

```bash
ls -la
```

This revealed several interesting files:

```text
Sup3rS3cretPickl3Ingred.txt
clue.txt
denied.php
login.php
portal.php
robots.txt
assets/
```

The most suspicious file was:

```text
Sup3rS3cretPickl3Ingred.txt
```

---

# 🥒 6. Ingredient #1

I attempted to read the file.

The application blocked the `cat` command and returned:

```text
Command disabled to make it hard for future PICKLEEEE RICCCKKKK.
```

Instead of using `cat`, I used another command capable of displaying the contents of a text file.

For example:

```bash
strings Sup3rS3cretPickl3Ingred.txt
```

This revealed the first ingredient:

```text
Mr. Meeseek Hair
```

### Ingredient #1

**Mr. Meeseek Hair** ✅

---

# 🔍 7. Following `clue.txt`

I then investigated:

```text
clue.txt
```

The file contained the following clue:

```text
Look around the file system for the other ingredient.
```

I used Linux filesystem enumeration to investigate other locations.

This eventually led me to the second ingredient.

### Ingredient #2

**1 Jerry Tear** ✅

---

# 👑 8. Privilege Escalation

At this stage I was still executing commands as:

```text
www-data
```

I checked the account's `sudo` permissions:

```bash
sudo -l
```

The important result was:

```text
User www-data may run the following commands:
    (ALL) NOPASSWD: ALL
```

This was a critical finding.

`NOPASSWD: ALL` meant that `www-data` could execute commands as other users, including `root`, without providing a password.

I verified this with:

```bash
sudo whoami
```

The result was:

```text
root
```

### Privilege Escalation Successful ✅

The privilege escalation path was:

```text
www-data
   ↓
sudo -l
   ↓
NOPASSWD: ALL
   ↓
sudo command
   ↓
root
```

> **Important note:** Because the command panel executes each submitted command separately, running `sudo su` did not make subsequent commands run as root. Using `sudo whoami` or another `sudo <command>` invocation executed the individual command as root.

---

# 🥒 9. Final Ingredient

After obtaining root-level command execution, I continued enumerating the filesystem to locate the final ingredient.

The final ingredient was:

```text
Fleeb Juice
```

### Ingredient #3

**Fleeb Juice** ✅

---

# 🏁 Final Results

| # | Ingredient | Status |
|---|---|---|
| 🥒 1 | Mr. Meeseek Hair | ✅ |
| 🥒 2 | 1 Jerry Tear | ✅ |
| 🥒 3 | Fleeb Juice | ✅ |

**Challenge Completed ✅**

---

# 🛠️ Tools Used

- Nmap
- Gobuster
- Linux CLI
- Browser Developer Tools
- Base64
- TryHackMe AttackBox

---

# 🧠 What I Learned

Since this was my first CTF, the biggest lesson was learning how individual pieces of information can be chained together.

The overall process was:

```text
Nmap
 ↓
Find web service
 ↓
Enumerate website
 ↓
Inspect source code
 ↓
Find credentials
 ↓
Login
 ↓
Discover command execution
 ↓
Obtain www-data
 ↓
Enumerate filesystem
 ↓
Find ingredient
 ↓
Follow clues
 ↓
Privilege escalation
 ↓
Root
 ↓
Find final ingredient
```

## Key lessons

### 1. Enumerate before exploiting

Instead of immediately attempting attacks, first understand what services and functionality are available.

### 2. Inspect the source code

The visible webpage is not necessarily the complete application.

HTML comments, hidden fields, and other source-code information can contain useful clues.

### 3. A blocked command does not necessarily mean the file is inaccessible

When `cat` was blocked, another Linux command could still be used to read the file.

### 4. Always check privileges

After obtaining command execution as `www-data`, checking:

```bash
sudo -l
```

revealed a direct privilege-escalation path.

### 5. Follow clues instead of guessing

The challenge provided clues that guided the next stage of enumeration.

### 6. Think in terms of an attack chain

A successful penetration test is often not one huge exploit.

It can be a chain of smaller discoveries:

```text
Information
    ↓
Enumeration
    ↓
Access
    ↓
More enumeration
    ↓
Privilege escalation
    ↓
Objective
```

### 7. Learn to work around restricted functionality

The web application blocked certain commands, such as `cat`. Instead of stopping, I considered alternative commands that could perform the same legitimate operation.

This reinforced the importance of understanding what a command actually does rather than simply memorizing command names.

---

# 💭 Personal Reflection

This was my **first Capture The Flag challenge**, and it helped me understand what practical cybersecurity work can look like beyond just learning commands and theory.

Before attempting this challenge, I had mainly been learning cybersecurity fundamentals through TryHackMe and had completed the OverTheWire Bandit wargame.

Pickle Rick was my first experience putting those concepts together against a complete target.

The biggest thing I learned was that penetration testing is not simply about memorizing tools or commands.

It is about:

> **Observe → Enumerate → Form a hypothesis → Test → Learn → Enumerate again.**

The challenge also showed me how a seemingly small discovery can become useful later in the attack chain.

For example:

```text
Username
   ↓
Password
   ↓
Portal
   ↓
Command execution
   ↓
www-data
   ↓
Filesystem enumeration
   ↓
sudo permissions
   ↓
Root
```

---

# 📚 Conclusion

Pickle Rick was a great introduction to practical penetration testing.

The challenge covered several fundamental concepts that I will continue building upon as I progress further into cybersecurity:

- Reconnaissance
- Enumeration
- Web security
- Authentication
- Command execution
- Linux permissions
- Privilege escalation

This challenge gave me my first practical experience taking a machine from initial reconnaissance to root-level access.

**First CTF completed. 🥒🔥**

---
