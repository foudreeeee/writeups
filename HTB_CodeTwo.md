# CTF – CodeTwo (Writeup)

## Context

In this challenge I face a web app that exposes a JavaScript code editor available after you create an account.  
The editor runs code server-side.  
The goal is to get initial access, then escalate privileges up to **root**.

---

## Recon

I start with a standard network scan to identify the exposed services:

    nmap -sC -sS -sV 10.10.11.82

The scan shows:
- an HTTP service on **port 8000**
- a web app that runs JavaScript through a built-in editor

After creating an account, I open the editor and confirm that the submitted code runs server-side.

---

## Initial access – Code execution via js2py

Looking at the environment, I find a known vulnerability in **js2py** that allows arbitrary code execution through a malicious JavaScript payload.

I set up a Netcat listener on my machine:

    nc -lvnp 4444

Then I inject a payload that exploits the js2py vulnerability straight into the code editor.  
Running the payload gives me a shell on the remote server.

To stabilize the shell:

    script /dev/null

---

## Grabbing sensitive data

Once on the target, I start a local HTTP server to expose the readable files:

    python3 -m http.server 8080

From my machine, I download a database found on the server:

    wget http://10.10.11.82:8080/users.db

---

## Extracting and cracking the credentials

Looking at the `users.db` database, I find user credentials, including an account named **marco** with a password hash.

I submit the hash to CrackStation, which recovers the plaintext password:

    sweetangelbabylove

I can then log in over SSH:

    ssh marco@10.10.11.82

---

## Local enumeration

Once logged in as `marco`, I grab the first flag (`user.txt`), then keep enumerating privileges.

I check the sudo rights:

    sudo -l

I see that `marco` can run a specific binary called **npbackup** with elevated privileges.

---

## Looking at npbackup

I print the binary's help to understand how it works:

    sudo npbackup --help

The binary runs backups based on a config file in **YAML** format.

That opens an abuse path through a user-controlled config.

---

## Privilege escalation via npbackup

I write a minimal malicious config file in `/tmp` that runs an arbitrary command during the backup.

When npbackup runs with this config, an executable `/tmp/rootbash` file is created.

I run it with `-p` to keep the privileges:

    /tmp/rootbash -p

I become **root** and grab the final flag in `/root/`.

---

## Conclusion

This challenge shows:
- the risk of running user code server-side,
- how known vulnerabilities (js2py) get exploited,
- the critical impact of credentials stored in accessible databases,
- the danger of binaries run through sudo that rely on controllable config files.

The exploitation combines **web RCE**, **data exfiltration**, **password cracking** and **privilege escalation** up to **root**.

---

## Skills demonstrated

- Network recon (Nmap)
- RCE through a JavaScript editor
- Reverse shell and stabilization
- File exfiltration (HTTP / wget)
- Database analysis
- Password cracking (hash)
- Local enumeration (sudo -l)
- Privilege escalation through a misconfigured sudo binary
