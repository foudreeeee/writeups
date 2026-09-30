# CTF – Environment (Writeup)

## Context

In this Hack The Box challenge, the target machine is reachable through the domain `environment.htb`.  
The goal is to get initial access to the web app, then escalate privileges up to **root**.

---

## Recon
I start with an nmap:

    nmap -sC -sV -sS IP
    
Then enumeration and fuzzing on the domain found:

    environment.htb

I quickly find a `/login` route. Inspecting the responses and some backend debug messages, I notice conditional logic tied to the app's runtime environment.

The comments say that:
- if the app runs in a **preprod** environment
- the user is automatically logged in as administrator (`user_id = 1`)

---

## Initial access – Auth bypass via an environment parameter

I intercept a login request and test adding an undocumented parameter:

    ?--env=preprod

This parameter forces the app to behave as if it were running in a preprod environment.  
Result:
- authentication is bypassed
- I'm logged in automatically to the portal as user **Hish**

This is a logic-based **auth bypass** rooted in poor environment handling.

---

## Finding an upload feature

Once logged in, I see that the portal lets you change the profile picture through an `/upload` endpoint.

I test the controls in place and find that:
- the MIME type is weakly checked
- files are stored in `/storage/files/`

---

## Initial access – Polyglot upload and command execution

I build a **polyglot** file:
- a valid PNG image
- that also carries PHP code

The app accepts the file and saves it in:

    /storage/files/

By hitting the uploaded file directly and adding a parameter like:

    ?cmd=<command>

the PHP code gets interpreted by the server.  
That lets me run arbitrary commands and get a **webshell**.

---

## Getting an interactive shell

From the webshell, I stabilize the access and get an interactive shell on the machine.

I'm logged in as:

    hish

---

## Local enumeration

I start by checking the sudo privileges:

    sudo -l

Interesting result:
- `hish` can run `/usr/bin/systeminfo` with `sudo`
- the **ENV** and **BASH_ENV** environment variables are preserved

That's a critical, exploitable point for privilege escalation.

---

## Privilege escalation – BASH_ENV abuse

The `systeminfo` binary is a shell script.  
When `BASH_ENV` is set, Bash loads and runs the referenced file automatically.

So I write a malicious script:

    /tmp/test.sh

Script contents:
- copy `/bin/bash` to `/tmp/rootbash`
- set the **SUID** bit

Then I run:

    sudo BASH_ENV=/tmp/test.sh /usr/bin/systeminfo

The script is interpreted with **root** privileges, which creates:

    /tmp/rootbash (SUID root)

---

## Root access

All that's left is to run:

    /tmp/rootbash -p

I get a **root** shell on the machine.

---

## Attack-chain summary

- Found the `--env=preprod` parameter
- Auth bypass (admin login)
- Upload of a PHP polyglot file
- Command execution through a webshell
- Shell access as `hish`
- `sudo` + `BASH_ENV` abuse
- Creating a SUID binary
- Getting a root shell

---

## Conclusion

This challenge shows:
- the dangers of poor environment handling (prod / preprod)
- the risk of insufficiently filtered uploads
- the critical impact of environment variables preserved under sudo
- why scripts run as root need a secure configuration

The exploitation combines:
- logic-based auth bypass
- RCE through file upload
- local privilege escalation
- abuse of internal shell mechanisms

---

## Skills demonstrated

- Web fuzzing and enumeration
- Application logic analysis
- Authentication bypass
- File upload exploitation
- Webshell and stabilization
- sudo enumeration
- Environment variable abuse (BASH_ENV)
- Linux privilege escalation
