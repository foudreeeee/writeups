# CTF – Principal (Writeup)

## Context

Linux box running a Java-style auth platform ("principal-platform") that issues encrypted JWTs (JWE) and publishes a JWKS endpoint. The flaw (CVE-2026-29000) is that the server trusts an unsigned `alg:none` JWT once it is wrapped in a JWE encrypted to the server's own public key. That gets an admin session; a service password from the dashboard gives SSH; and a Dirty-Pipe-style overwrite of `/usr/bin/su` gets root.

## Recon

No nmap file in my notes. The one surface that mattered:

- A web app on **8080** with a JWKS endpoint at `/api/auth/jwks` and a token-gated `/dashboard`. The front end stores its bearer token in `sessionStorage` under `auth_token`.

## Initial access – JWE-wrapped PlainJWT forgery (CVE-2026-29000)

The server accepts a JWE whose inner payload (`cty: JWT`) is a plain, unsigned JWT (`alg: none`). Because the JWE is encrypted to the server's *public* key — which the server itself publishes at `/api/auth/jwks` — anyone can produce a valid JWE, and the server never checks a signature on the inner token. So I can mint an admin token.

`poc.py` fetches the public key from JWKS, builds an `alg:none` JWT for `admin` / `ROLE_ADMIN`, and encrypts it into a JWE (`RSA-OAEP-256` / `A128GCM`, `cty: JWT`):

```
python3 poc.py \
  --jwks http://target:8080/api/auth/jwks \
  --user admin \
  --role ROLE_ADMIN
```

The inner claims it forges:

```json
{"sub":"admin","role":"ROLE_ADMIN","iss":"principal-platform","iat":...,"exp":...}
```

I take the resulting token and inject it into the app's session from the browser console:

```js
sessionStorage.setItem("auth_token", "PASTE_TOKEN_HERE")
```

Then I browse to `/dashboard` as admin.

## Loot – svc-deploy password → SSH

The dashboard settings page exposes a password for the `svc-deploy` account. That works over SSH:

```
ssh svc-deploy@principal.htb
```

User flag is in `svc-deploy`'s home.

## Privilege escalation – Dirty-Pipe overwrite of setuid /usr/bin/su

The intended local privesc (per my notes) was writing an SSH key into a `.ssh` directory writable by a group `svc-deploy` belonged to, planting a key for a higher-privileged account. I instead used a page-cache overwrite, and it worked.

`poc2.py` is a Dirty-Pipe-style exploit (CVE-2022-0847 technique). It:

1. Opens the setuid-root `/usr/bin/su` read-only.
2. Sets up an `AF_ALG` socket (`socket(38, 5, 0)`) and uses `splice()` to pull `su`'s bytes into a pipe, then writes back into the page cache through the socket — the same "pipe page merge" primitive Dirty Pipe uses to modify a file you only have read access to.
3. Writes a small patched blob (zlib-compressed in the script) over `su` in 4-byte chunks, so that invoking `su` spawns a root shell.
4. Calls `su`.

```
python3 poc2.py
# lands a root shell via the patched su
```

Root.

## Conclusion

The auth service trusted an unsigned JWT as long as it was encrypted to its own published public key (CVE-2026-29000), so forging an admin JWE and dropping it into `sessionStorage` gave an admin dashboard. The dashboard leaked the `svc-deploy` SSH password. Root was a Dirty-Pipe-style overwrite of the setuid `/usr/bin/su` binary.

## Skills demonstrated

- JWE/JWT confusion: wrapping an `alg:none` token in a JWE encrypted to a public JWKS key (CVE-2026-29000)
- Session hijack by injecting a forged token into `sessionStorage`
- Harvesting service credentials from an authenticated dashboard
- Dirty-Pipe page-cache overwrite of a setuid binary (`/usr/bin/su`) for root (CVE-2022-0847 technique)
