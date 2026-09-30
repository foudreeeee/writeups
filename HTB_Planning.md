# CTF – Planning (Writeup)

## Context

In this Hack The Box challenge, the target machine is reachable at `10.10.11.68` under the domain `planning.htb`.  
Initial credentials are provided, but the main site on port 80 isn't reachable.

The goal is to get initial access through an exposed service, then escalate privileges up to **root**.

---

## Initial information

Credentials provided at the start:

- User: admin  
- Password: 0D5oT70Fq13EvB5r  

---

## Recon

I start by testing access to the website on port 80, with no luck.  
So I decide to look for subdomains tied to `planning.htb`.

I run DNS fuzzing with ffuf:

    ffuf -w /usr/share/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt \
         -u http://10.10.11.68/ \
         -H "Host: FUZZ.planning.htb" \
         -fs 178

This fuzzing lets me find the following subdomain:

- `grafana.planning.htb`

---

## Reaching Grafana

Hitting `http://grafana.planning.htb`, I find a **Grafana** interface.

I try the initial credentials that were provided:

- admin / 0D5oT70Fq13EvB5r  

The login **succeeds**, which gives me authenticated access to the Grafana interface.

---

## Initial access – Grafana exploitation

After identifying the version, I see Grafana is version **11.0**, known to be vulnerable to a flaw that allows arbitrary command execution with valid credentials.

I find a Python script on GitHub that exploits this vulnerability.  
Using it, I send a command that opens a **reverse shell** back to my machine.

I get a shell on the target.

---

## Post-exploitation – Environment variables

Once connected, I run the following to inspect the environment variables:

    env

I find credentials stored in cleartext:

- User: enzo  
- Password: RioTecRANDEntANT!  

---

## SSH user access

I test these credentials over SSH right away:

    ssh enzo@10.10.11.68

The login succeeds.  
I grab the **user flag** (`user.txt`).

---

## Local enumeration

I try to enumerate the sudo privileges:

    sudo -l

This command isn't allowed for `enzo`.  
So I run **linPEAS** to find other escalation vectors.

The enumeration surfaces an interesting binary:

- `/tmp/bash`

---

## Privilege escalation

The `/tmp/bash` binary has the **SUID** bit set.  
I run it with `-p` to keep the privileges:

    /tmp/bash -p

This command gives me a shell with **root** privileges right away.

I can then read the file:

    /root/root.txt

and grab the **root flag**.

---

## Alternative note

If the `/tmp/bash` binary hadn't been there, another path would have worked:  
a **cron service** running in a loop with root privileges.

In that scenario, it would have been possible to:
- exploit the cron
- set up a tunnel
- copy `/bin/bash`
- and set the SUID bit to get a root shell

---

## Attack-chain summary

- No initial access on port 80
- Found the `grafana.planning.htb` subdomain
- Grafana authentication with provided creds
- Grafana 11.0 exploitation (RCE)
- Reverse shell
- Credential recovery through environment variables
- SSH user access
- Found a SUID binary (`/tmp/bash`)
- Privilege escalation and root access

---

## Conclusion

This challenge shows:
- why subdomain fuzzing matters
- the risk of exposed admin services
- the danger of credentials stored in cleartext in environment variables
- the critical impact of SUID binaries left accessible

The exploitation combines:
- web recon
- exploiting a vulnerable third-party service
- post-exploitation
- local privilege escalation up to **root**

---

## Skills demonstrated

- DNS fuzzing (ffuf)
- Web recon
- Grafana exploitation
- Reverse shell
- Environment variable analysis
- SSH access
- Linux local enumeration
- SUID binary exploitation
- Privilege escalation
