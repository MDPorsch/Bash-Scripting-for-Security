# Bash Scripting for Security
### Cybersecurity Homelab Series

---

##  Overview

**Date:** March 5, 2026
**Environment:** Ubuntu VM (VirtualBox on MacBook M4)
**Difficulty:** Beginner
**Time Taken:** ~60 minutes

In my previous exercise we got comfortable with the Linux terminal. In this exercise I level up, instead of typing commands one by one, we write scripts that automate them. This is how real security professionals work. Whether you're a SOC analyst automating log checks, a pentester writing recon scripts, or a security engineer building monitoring tools, bash scripting is a core skill you'll use every single day.

---

##  Objectives

- Understand what a bash script is and how to write one
- Use variables, commands and output redirection in scripts
- Write a system information gatherer
- Write a log monitor that scans for suspicious activity
- Write a system health checker
- Debug real errors and fix them

---

##  Tools & Environment

| Item | Detail |
|------|--------|
| Host Machine | MacBook M4 |
| VM Platform | VirtualBox |
| Operating System | Ubuntu (Linux) |
| Terminal | Bash |
| Editor | Nano |
| Working Directory | ~/lab/ex02 |

---

##  Step-by-Step Walkthrough

### Step 1 — Setting Up the Workspace

```bash
cd ~ && mkdir -p ~/lab/ex02 && cd ~/lab/ex02 && pwd
```

Output: `/home/mdubuntu/lab/ex02`

Good housekeeping, every exercise gets its own folder so work stays organised.

---

### Step 2 — Understanding Bash Scripts

A bash script is just a text file containing terminal commands. Instead of typing them one by one, you put them all in a file and run it in one shot.

Every bash script starts with a **shebang:**
```bash
#!/bin/bash
```
This tells Linux "use bash to run this file." Without it, Linux won't know how to interpret the file.

Other key concepts used throughout this exercise:

| Syntax | Meaning |
|--------|---------|
| `#` | Comment — Linux ignores this line |
| `echo` | Print text to the screen |
| `$(command)` | Run a command and use its output inline |
| `grep "text" file` | Search a file for lines containing text |
| `\|` (pipe) | Feed output of one command into another |
| `tail -5` | Show only the last 5 lines |
| `chmod +x` | Make a file executable |
| `./script.sh` | Run a script from the current directory |

---

### Step 3 — Script 1: System Info Gatherer

```bash
nano myfirstscript.sh
```

```bash
#!/bin/bash

# My first bash script
# This script tells us basic system information

echo "===== System Info ====="
echo "Hostname: $(hostname)"
echo "Current User: $(whoami)"
echo "Current Date: $(date)"
echo "Uptime: $(uptime -p)"
echo "===== Done ====="
```

Made it executable and ran it:

```bash
chmod +x myfirstscript.sh && ./myfirstscript.sh
```

Output:
```
===== System Info =====
Hostname: mdubuntu
Current User: mdubuntu
Current Date: Thu Mar  5 14:38:22 WAT 2026
Uptime: up 7 hours, 20 minutes
===== Done =====
```



The `$(command)` syntax is powerful — it runs any command and drops the result directly into the output. So `$(whoami)` gets replaced by the actual username at runtime. This is called **command substitution.**

> **Why this matters:** After compromising a system, attackers run automated recon scripts to gather system info instantly. Defenders write the same kind of scripts for asset inventory and monitoring. This is the foundation of both.

---

### Step 4 — Script 2: Log Monitor

This script scans `/var/log/auth.log` — the file that records every login attempt, sudo usage, and user session on the system.

```bash
nano logmonitor.sh
```

```bash
#!/bin/bash

# Log Monitor Script
# Scans system logs for suspicious activity

echo "===== Log Monitor ====="
echo "Running as: $(whoami)"
echo "Date: $(date)"
echo ""

echo "--- Failed Login Attempts ---"
grep "authentication failure\|Failed" /var/log/auth.log | tail -5

echo ""
echo "--- Successful Logins ---"
grep "session opened" /var/log/auth.log | grep -v "CRON" | tail -5

echo ""
echo "--- Cron Activity ---"
grep "CRON" /var/log/auth.log | tail -5

echo ""
echo "--- Sudo Usage ---"
grep "sudo" /var/log/auth.log | tail -5

echo ""
echo "--- New User Accounts Created ---"
grep "new user" /var/log/auth.log | tail -5

echo ""
echo "===== Done ====="
```

```bash
chmod +x logmonitor.sh && ./logmonitor.sh
```

Output:
```
===== Log Monitor =====
Running as: mdubuntu
Date: Thu Mar  5 14:58:03 WAT 2026

--- Failed Login Attempts ---

--- Successful Logins ---
2026-03-05T14:27:19 mdubuntu pkexec: session opened for user root by mdubuntu
2026-03-05T14:51:03 mdubuntu sudo: session opened for user root by mdubuntu

--- Cron Activity ---
2026-03-05T14:57:01 mdubuntu CRON[24174]: session opened for user mdubuntu
2026-03-05T14:58:01 mdubuntu CRON[24218]: session opened for user mdubuntu

--- Sudo Usage ---
2026-03-05T14:51:03 mdubuntu sudo: mdubuntu TTY=/dev/pts/0 ; COMMAND=/usr/bin/cat /var/log/auth.log

--- New User Accounts Created ---

===== Done =====
```



Key observations from the output:

- **No failed logins** — system is clean, nobody has tried to brute force it
- **Sudo usage logged** — you can see exactly which command was run, from which directory, at what time
- **Cron jobs from Exercise 1 visible** — everything leaves a trace in auth.log
- **The script logged itself being run** — meta! The `cat /var/log/auth.log` command from earlier showed up as a sudo entry

> **Why this matters:** This is exactly what a SOC analyst runs during an investigation. `auth.log` tells the full story of who did what on a system. Failed login spikes = brute force attack. Unexpected sudo commands = possible privilege escalation. New user creation at 3am = backdoor account.

---

### Step 5 — Script 3: System Health Checker

The most comprehensive script — a full security overview of the system in one shot.

```bash
nano healthcheck.sh
```

```bash
#!/bin/bash

# System Health Check Script
# Gives a quick security overview of the system

echo "==============================="
echo "   SYSTEM HEALTH CHECK"
echo "   $(date)"
echo "==============================="
echo ""

echo "--- Disk Usage ---"
df -h | grep -v tmpfs

echo ""
echo "--- Memory Usage ---"
free -h

echo ""
echo "--- Logged In Users ---"
who

echo ""
echo "--- Last 5 Logins ---"
journalctl _SYSTEMD_UNIT=systemd-logind.service | tail -5

echo ""
echo "--- Listening Network Ports ---"
ss -tuln

echo ""
echo "--- Top 5 CPU Processes ---"
ps aux --sort=-%cpu | head -6

echo ""
echo "==============================="
echo "   HEALTH CHECK COMPLETE"
echo "==============================="
```

```bash
chmod +x healthcheck.sh && ./healthcheck.sh
```



**Breaking down the key findings:**

**Disk Usage:**
```
/dev/sda2   34G   16G   17G   49%
```
VM is half full at 49% — worth monitoring as more tools get installed.

**Memory:**
```
Total: 4.8GB    Used: 2.6GB    Free: 970MB
Swap: 1.3GB in use
```
Swap memory is being used which means RAM is under pressure. Closing Firefox during lab work frees up significant memory.

**Last 5 Logins:**
```
Mar 04 13:17:42 — New session of user labuser
Mar 04 13:21:06 — labuser session logged out
```
The `labuser` account created in Exercise 1 shows up in the login history! Every action across exercises is connected and traceable.

**Open Network Ports:**
```
tcp  LISTEN  0.0.0.0:22   ← SSH open on all interfaces
tcp  LISTEN  0.0.0.0:631  ← CUPS printing service
tcp  LISTEN  127.0.0.53:53 ← DNS resolver (local only)
```
Port 22 (SSH) is listening — meaning any machine on the network could attempt to connect. This will be hardened in Exercise 4.

**Top CPU Processes:**
```
gnome-shell   2.1%  ← Desktop environment
suricata      0.8%  ← Network IDS actively monitoring
ptyxis        0.6%  ← Terminal emulator
```
Suricata (the network intrusion detection system) is actively running and consuming resources — it's watching all network traffic in real time.

---

##  Mistakes & Fixes

### Mistake 1 — `last` command not found
**What happened:** The healthcheck script used `last` to show recent logins but Ubuntu returned:
```
./healthcheck.sh: line 25: last: command not found
```
**What I tried:** Attempted to install via `sudo apt install util-linux` — already at newest version.

**Fix:** Used `journalctl _SYSTEMD_UNIT=systemd-logind.service | tail -5` instead, which pulls login history directly from systemd's journal.

**Lesson:** Commands vary between Linux distributions and versions. When something isn't available, check the system's journal with `journalctl` — it logs almost everything.

### Mistake 2 — Duplicate lines in logmonitor.sh
**What happened:** When editing the script mid-exercise, old lines weren't fully removed, causing section headers to print twice.

**Fix:** Opened nano, cleared the entire file with `Ctrl + K` and pasted a clean version.

**Lesson:** Always review your full script before running it. A quick `cat scriptname.sh` before executing saves debugging time.

---

##  Key Takeaways

1. **Bash scripts = automation** — any commands you type manually can be scripted and run in one shot
2. **`$(command)` is powerful** — embed live command output directly into your scripts
3. **`grep` + `|` (pipe) is your best friend** — search and filter any file or output instantly
4. **`auth.log` tells the full story** — every login, sudo command and session is logged there
5. **Open ports = attack surface** — always know what's listening on your system
6. **Commands differ across distros** — when something fails, troubleshoot and find the equivalent

---

##  Scripts Written This Exercise

| Script | Purpose |
|--------|---------|
| `myfirstscript.sh` | System info gatherer |
| `logmonitor.sh` | Auth log scanner for suspicious activity |
| `healthcheck.sh` | Full system health and security overview |

---

##  Real-World Relevance

| Skill Practiced | Real-World Application |
|----------------|----------------------|
| Bash scripting | Automating security tasks, writing tools |
| Log monitoring | SOC analysis, incident response |
| Port scanning own system | Attack surface awareness |
| Process monitoring | Threat hunting, malware detection |
| Troubleshooting errors | Real-world debugging skills |

These three scripts form the basis of a real **security monitoring toolkit.** With small additions they could run on a schedule via cron (Exercise 1!) and email alerts when suspicious activity is detected — that's essentially what enterprise SIEM tools do at their core.

---



---

**Connect with me:** [Your LinkedIn] | [Your GitHub] | [Your Twitter/X]
