# Cap — Hack The Box

**Tags:** [privilege escalation], [IDOR], [setuid], [Linux capabilities]

**Target IP:** 10.129.32.33

---

## Recon

The first step I took on this machine was basic recon. I ran `nmap 10.129.32.33` to search for open ports on the machine. This command returned ports 21, 22, and 80.

- Port 21 — FTP
- Port 22 — SSH
- Port 80 — HTTP (unencrypted, packets can be intercepted)

After checking for basic ports, I ran `nmap -Pn -p- --min-rate 10000 10.129.32.33` to check for all ports in case there were nonstandard ports open. There were not.

After this, I ran `nmap -Pn -p 21,22,80 -sV -sC 10.129.32.33` which enumerates details on open ports, software versions, etc. This returned the following important information: FTP running vsftpd 3.0.3, SSH on OpenSSH 8.2p1, and a web app called "Security Dashboard" on port 80 behind gunicorn.

## Enumeration

I entered the IP address into the URL bar, which led me to a website called Security Dashboard, as indicated by the title tag that our recon scan revealed. I navigated to the Security Snapshot page, which showed basic security events. The URL bar displayed `http://10.129.32.33/data/4`, which indicated that the `4` could be altered — an **IDOR** (Insecure Direct Object Reference) vulnerability.

I tested this starting with `1`, which revealed a security dashboard for a different user. I was able to download the pcap file using the dropdown menu on the right side of the GUI, and analyzed it using Wireshark. I followed this process for all users 0–9, eventually locating sensitive information transferred via FTP from user 0.

The sensitive information was user Nathan's password. The password worked for both FTP login and SSH login.

## Initial Access

I used SSH to gain initial access utilizing the following command:

```
ssh nathan@10.129.32.33
```

I entered his password when prompted. The user flag was located in Nathan's home directory (`/home/nathan/user.txt`).

## Privilege Escalation

My next step was to find a binary that would allow me to execute commands as root.

I initially ran `find / -perm -4000 -type f 2>/dev/null`, a one-liner that searches for SUID binaries. However, nothing of use was returned, so I ran `getcap -r / 2>/dev/null`, which searches for files with Linux capabilities — sometimes granting powerful root-level permissions for specific processes.

I found the following:

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

The `cap_setuid` capability grants the ability to change the process user ID, effectively allowing escalation to root.

I utilized this to spawn a shell with root permissions:

```
python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

This spawned a shell with root privileges. I then navigated to `/root/` and printed the root flag (`/root/root.txt`).
