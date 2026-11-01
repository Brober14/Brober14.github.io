---
title: Silent Monitor
date: 2026-11-01
difficulty: TBD
os: Linux
summary: A SQL injection bypasses the login, command injection in the Host Health page leaks the sysadmin credentials, and a crackable KeePass database in the backups folder gives root.
room: TBD
---

## Recon

I started with an nmap scan of the target. It found SSH open on port 22 and a Werkzeug web server on port 5050.

![nmap scan](writeups/img/silentmonitor-1.png)

The webpage didn't show anything of interest, so I ran a couple of enumeration rounds with gobuster. They found an `/internal` login page. I kept looking for anything else of interest, but found nothing.

![gobuster scan](writeups/img/silentmonitor-2.png)

## Foothold

### Dashboard access

I spent some time searching online for vulnerabilities that might be exploitable in this version, but found nothing of use. I then returned to the login page, as it was the only point of interest I had found, and started looking for a way in. After looking at how the login form works, I ruled out brute-force because I had no valid login ID. On a whim, I tried the most common type of SQL injection, `' OR 1=1; --`, and it worked. I was in. I couldn't believe that the way in was just the most common SQL injection.

![Dashboard behind the login page](writeups/img/silentmonitor-3.png)

### Remote code execution

I started looking around the dashboard for anything interesting, and one page caught my attention: Host Health, which sends ping requests to a provided target IP to check whether it is alive.

![Host Health page](writeups/img/silentmonitor-4.png)

The flow is very simple: you provide an IP, the web host pings it to see whether it responds, and the result of the command is returned to you.

![Ping result for 127.0.0.1](writeups/img/silentmonitor-5.png)

That meant I could try to inject additional commands into the request. I intercepted the request with Burp and added `%0a[cmd]` to the target, which got me remote code execution on the host. This is our way in.

In the example below, the target was modified to `IP%0als`. The response contained a listing of the shell's current directory, so it was a success.

![Command injection with ls](writeups/img/silentmonitor-6.png)

### SSH as sysadmin

The `ls` output listed an interesting file called `secret.config`. Reading it revealed the credentials of a backup service account called `sysadmin`.

![secret.config](writeups/img/silentmonitor-7.png)

I tried the credentials over SSH, and they worked.

![SSH login as sysadmin](writeups/img/silentmonitor-8.png)

The user flag was in the home directory of `sysadmin`. From there, I needed to escalate my privileges to gain root access.

![sysadmin home directory](writeups/img/silentmonitor-9.png)

## Privilege escalation

### sysadmin → root

Escalation to root was actually the easiest part of this room. The `backups` folder contained a KeePass database file, which I downloaded and converted to a hash with `keepass2john`. I then ran a John brute-force attack against it and found the password for the database.

![John cracking the KeePass hash](writeups/img/silentmonitor-10.png)

The database contained a single entry: the password for the root user. I used it to become root and read the final flag.

![KeePass entry](writeups/img/silentmonitor-11.png)

## Flags

user.txt — THM{hidden}
root.txt — THM{hidden}
