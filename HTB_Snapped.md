# CTF – Snapped (Writeup)

## Context
Snapped is a Linux box exposing an admin panel (`admin.snapped.htb`) powered by
Nginx UI. The management version in use (`2.3.2`) has an unauthenticated backup
endpoint that also discloses the encryption key, which unravels the whole box.

## Recon
Directory brute forcing against the admin vhost surfaced the Nginx UI panel and its
version:

```
gobuster dir -u http://admin.snapped.htb/ -w ~/SecLists/Discovery/Web-Content/common.txt
```

The disclosed version is vulnerable to an unauthenticated data-exfiltration flaw on
its backup API.

## Initial access – unauthenticated backup download + key disclosure
Nginx UI's `/api/backup` endpoint can be hit without authentication. It returns an
encrypted archive **and** hands back the AES-256 key and IV in the response header
`X-Backup-Security` (formatted `base64_key:base64_iv`). `poc.py` automates download
and decryption:

```
python poc.py --target http://admin.snapped.htb:9000 --out backup.bin --decrypt
```

The script parses the header, verifies the 32-byte key / 16-byte IV, and decrypts each
member with AES-256-CBC (PKCS#7):

```python
cipher = AES.new(base64.b64decode(key), AES.MODE_CBC, base64.b64decode(iv))
unpad(cipher.decrypt(encrypted_data), AES.block_size)
```

The archive contained `hash_info.txt`, `nginx.zip` and `nginx-ui.zip`. Decrypted,
`nginx-ui.zip` yielded:

- `app.ini` — the Nginx UI config, leaking the `JwtSecret`
  (`6c4af436-035a-4942-9ca6-172b36696ce9`), `node.Secret`, and the app
  `crypto.Secret`.
- `database.db` — the Nginx UI SQLite database containing bcrypt user hashes.

`hash_info.txt` confirmed the backup contents and the Nginx UI version (`2.3.2`).

## Cracking – bcrypt → SSH credentials
The user hashes pulled from `database.db` (`bcrypt_hashes.txt`):

```
admin     $2a$10$8YdBq4e.WeQn8gv9E0ehh.quy8D/4mXHHY4ALLMAzgFPTrIVltEvm
jonathan  $2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq
```

Cracking `jonathan`'s hash gave `linkpark`. Those credentials were reused for SSH,
landing a shell as `jonathan`.

## Privilege escalation – kernel LPE
`linpeas.sh` and two PoC scripts staged on the box point at a local kernel exploit.

- `poc2.py` — labeled `CVE-2025-38236 Full PoC - af_unix UAF`. It sprays abstract
  `AF_UNIX` sockets to groom the heap, then repeatedly sends normal and `MSG_OOB`
  data across a connected pair to trigger a use-after-free in the AF_UNIX out-of-band
  handling, checking `os.geteuid() == 0` after each attempt.
- `poccopyfail.py` — an alternate attempt (the name flags it as the one that failed):
  it opens an `AF_ALG` socket (`authencesn(hmac(sha256),cbc(aes))`), then abuses
  `sendmsg` + `splice()` to write attacker-controlled bytes into `/usr/bin/su` before
  invoking `su`, i.e. an on-disk binary overwrite path.

The working path was the AF_UNIX MSG_OOB use-after-free (`poc2.py`), which yields a
root shell after heap grooming.

> The provided notes stop at the `jonathan:linkpark` foothold; the privilege-
> escalation step above is reconstructed from the staged exploit scripts.

## Conclusion
An unauthenticated Nginx UI backup endpoint leaked both the encrypted backup and its
AES key in a response header. Decrypting it exposed config secrets and the user
database; cracking `jonathan`'s bcrypt hash (`linkpark`) gave SSH access. A local
kernel use-after-free in AF_UNIX (CVE-2025-38236) escalated to root.

## Skills demonstrated
- Web directory enumeration and version fingerprinting (Nginx UI)
- Exploiting an unauthenticated backup endpoint with in-band key disclosure
- AES-256-CBC decryption of exfiltrated archives
- Parsing secrets and SQLite databases from recovered backups
- Offline bcrypt cracking and credential reuse for SSH
- Kernel LPE via an AF_UNIX MSG_OOB use-after-free (CVE-2025-38236)
