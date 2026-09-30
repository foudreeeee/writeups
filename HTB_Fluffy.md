# CTF – Fluffy (Writeup)

## Context

In this **Windows / Active Directory** Hack The Box challenge, the target machine is reachable at `10.10.11.69` under the domain `fluffy.htb`.  
Initial credentials are provided, which let me start enumerating the domain internally.

The goal is to get user access, then escalate privileges step by step up to **Domain Administrator**.

---

## Initial recon

I start with an Nmap scan to identify the exposed services:

    nmap -sC -sV 10.10.11.69

The results show a typical Windows environment with:
- LDAP
- SMB
- Kerberos
- Active Directory services

Initial credentials are provided:

- User: `j.fleischman`
- Password: `J0elTHEM4n1990!`

---

## SMB enumeration

I start by listing the accessible SMB shares:

    smbclient -L 10.10.11.69 -U "fluffy.htb/j.fleischman"

Among the available shares, the **IT** share stands out.  
I test access:

    smbclient \\10.10.11.69\IT -U "fluffy.htb/j.fleischman"

Result:
- **read / write** access allowed

That's a critical attack surface.

---

## Initial access – Exploiting CVE-2025-24071 (NTLM Leak)

Since the IT share is writable, I exploit **CVE-2025-24071**.

How it works:
- upload a booby-trapped `.library-ms` file
- paired with a `.zip` archive
- when the victim interacts with the file, an outbound NTLM authentication is triggered

I drop the malicious files on the IT share and launch **Responder** on my attacking machine.

Result:
- an **NTLMv2** hash leaks
- compromised user: `p.agila`

---

## Cracking the hash and new access

I crack the recovered NTLMv2 hash, which gives me the password:

- User: `p.agila`
- Password: `prometheusx-303`

I now have new valid access to the domain.

---

## Active Directory enumeration (BloodHound)

With `p.agila`'s credentials, I run an Active Directory enumeration with **BloodHound**.

Key finding:
- `p.agila` can add itself to the **SERVICE ACCOUNTS** group
- that group has **GenericWrite** rights over several service accounts:
  - `ca_svc`
  - `ldap_svc`
  - `winrm_svc`

That opens the door to an advanced attack through **Shadow Credentials**.

---

## Shadow Credentials abuse (Certipy)

I start by adding `p.agila` to the **SERVICE ACCOUNTS** group.

Then I use **Certipy** to exploit **Shadow Credentials** on the `winrm_svc` account.

How it works:
- inject a Kerberos key into the AD object of `winrm_svc`
- recover the account's **NT hash**

With that hash, I can connect remotely through **Evil-WinRM**.

---

## User access through Evil-WinRM

I connect successfully as `winrm_svc` and grab the **first user flag**.

At this point I have interactive access on the machine.

---

## Analyzing AD Certificate Services (AD CS)

I keep enumerating and analyze the **Active Directory Certificate Services** infrastructure.

I find a critical vulnerability:
- **ESC16** on the certificate authority:
  - `fluffy-DC01-CA`

The `p.agila` account has inherited rights over `ca_svc`, which makes the exploitation possible.

---

## ESC16 exploitation – Certificate abuse

Attack steps:

1. I temporarily change the UPN of `ca_svc` to set it to:
   
       administrator

2. I request a certificate through a valid client-authentication template

3. Once I have the certificate, I immediately restore `ca_svc`'s original UPN to limit visible traces

---

## Domain Administrator access

With the recovered certificate, I use:

    certipy auth

This lets me:
- get a Kerberos TGT
- recover the **domain administrator's NT hash**

I now have full **Domain Admin** access.

I can then read the **root flag**.

---

## Attack-chain summary

- Initial access with provided credentials
- SMB enumeration (writable IT share)
- CVE-2025-24071 exploitation (NTLM leak)
- Cracking the NTLMv2 hash
- AD enumeration with BloodHound
- GenericWrite abuse on service accounts
- Shadow Credentials exploitation (Certipy)
- WinRM access
- AD CS exploitation (ESC16)
- Domain Administrator compromise

---

## Conclusion

This challenge shows:
- the risk of misconfigured SMB shares
- the critical impact of NTLM leaks
- how dangerous poorly delegated AD rights are
- the power of modern attacks against AD CS

The exploitation combines realistic, current and advanced Active Directory attack techniques.

---

## Skills demonstrated

- SMB and Active Directory enumeration
- NTLM exploitation (Responder)
- NTLMv2 hash cracking
- AD graph analysis (BloodHound)
- AD permission abuse (GenericWrite)
- Shadow Credentials (Certipy)
- AD CS attacks (ESC16)
- WinRM access and Windows post-exploitation
- Domain Admin compromise
