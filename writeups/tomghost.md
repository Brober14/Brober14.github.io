---
title: Tomghost
date: 2026-10-16
difficulty: Easy
os: Linux
summary: Ghostcat (CVE-2020-1938) leaks SSH credentials from web.xml; a cracked PGP file gives merlin, and sudo zip gives root.
room: https://tryhackme.com/room/tomghost
---

## Recon

I added the target's IP to `/etc/hosts` as `tomghost.thm` and started with a basic nmap scan of open ports:

```bash
nmap -sV -sC tomghost.thm -oN basic.nmap
```

![nmap scan](writeups/img/tomghost-1.png)

The scan showed SSH on port 22 and HTTP on port 8080, plus ports 8009 and 53, which I wasn't familiar with yet. Nmap identified 8009 as Apache JServ, while 53 came back as tcpwrapped with no details about what was running behind it.

Since port 8080 was running Apache Tomcat, I opened the website in the browser to see what was on it. It turned out to be the standard template page you get after installing Tomcat.

![Tomcat webpage](writeups/img/tomghost-2.png)

At the same time, I ran gobuster against the site to enumerate URL paths and see if anything popped up:

```bash
gobuster dir -u http://tomghost.thm:8080/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x html,js,txt,pdf
```

![gobuster scan](writeups/img/tomghost-3.png)

Nothing stood out, only the usual paths for the template page. I checked `/manager`, but it's protected, and I couldn't get in without local access or authorisation.

While gobuster was running, I looked up known vulnerabilities for Apache Tomcat 9.0.30 and found CVE-2020-1938, also known as Ghostcat. The challenge page mentions that name, so this looked like the way in. It's also rated 9.8 (critical), which made it a great find. I read up on what the vulnerability is and how to exploit it.

## Foothold

I searched msfconsole for ghostcat and, luckily, it had exactly what I was looking for. There's an auxiliary module that reads files from the server by exploiting Tomcat's AJP trust issue.

![msfconsole search results](writeups/img/tomghost-4.png)

I loaded the module, read the extra info on how it works and checked which options it needed. It only takes the remote port, the remote IP and the file to read. Two of them were already filled in, so I just had to set the target IP to test whether the exploit would work here. I set it and typed `run`. A few seconds later I had the web.xml file from the remote host and, to my surprise, that was all it took. The credentials to get onto the machine were sitting right there with the rest of the data.

![msfconsole exploit results](writeups/img/tomghost-5.png)

I used them to log in as skyfuck over SSH:

```bash
ssh skyfuck@<TARGET_IP>
# password: [redacted]
```

## Privilege escalation

### skyfuck → merlin

Once I was in as skyfuck, I looked around and found a GPG-encrypted file in the home directory, along with the private key for it.

![Encrypted file and private key](writeups/img/tomghost-6.png)

I downloaded the file, converted it to a hash with gpg2john and cracked the passphrase with john. Then I decrypted the file on the target, which gave me the credentials for merlin. I switched to merlin and found the user.txt flag in his home directory.

![merlin user flag](writeups/img/tomghost-7.png)

### merlin → root

Next, I looked for privilege escalation paths and found that merlin can run the zip binary as root with sudo. I checked GTFOBins for zip and found a command that spawns a shell:

```bash
sudo zip /tmp/f /etc/hosts -T -TT '/bin/bash #'
```

Because the command runs through sudo, the shell it spawns is a root shell. From there I read the root.txt flag, which finished the challenge.

![merlin to root](writeups/img/tomghost-8.png)

## Flags

user.txt — THM{hidden}
root.txt — THM{hidden}
