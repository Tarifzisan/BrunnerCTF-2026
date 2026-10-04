# CleanDesk — Writeup

**Category:** Mobile
**Difficulty:** Easy-Medium
**Author:** KyootyBella

## Challenge Description

> A colleague left Brunnerne A/S on a Friday, and the handover was shorter than anyone would have liked. IT imaged her phone before wiping it, which is policy, and filed the image on the shared drive, which is not.
>
> Every app on that device encrypts its data at rest. IT confirmed this in the offboarding ticket and closed it.

We are given a single file:

```text
cleandesk.ab
```

The goal is to recover the hidden flag.

---

## 1. Identifying the File

First, determine what type of file we have:

```bash
file cleandesk.ab
```

Output:

```text
cleandesk.ab: Android Backup, version 5, Compressed, Not-Encrypted
```

The important part is:

```text
Not-Encrypted
```

The challenge says that the applications encrypt their data at rest, but that does **not** necessarily mean the Android backup container itself is encrypted.

The `.ab` file is an **Android ADB Backup**.

A version 5 backup has a structure similar to:

```text
ANDROID BACKUP
5
1
none
<compressed data>
```

The final part is a zlib-compressed TAR archive.

---

## 2. Extracting the Backup

We can manually remove the four-line header and decompress the remaining data.

```python
import zlib

with open("cleandesk.ab", "rb") as f:
    data = f.read()

pos = 0

# Skip the four-line Android Backup header
for _ in range(4):
    pos = data.index(b"\n", pos) + 1

tar_data = zlib.decompress(data[pos:])

with open("cleandesk.tar", "wb") as f:
    f.write(tar_data)
```

Then extract the TAR archive:

```bash
tar -xf cleandesk.tar
```

List the extracted files:

```bash
find . -type f
```

We get:

```text
apps/dk.brunnerne.onevoice/_manifest
apps/dk.brunnerne.onevoice/sp/com.google.android.gms.measurement.prefs.xml
apps/dk.brunnerne.onevoice/sp/session.xml
apps/dk.brunnerne.onevoice/sp/dk.brunnerne.onevoice_preferences.xml
apps/dk.brunnerne.onevoice/sp/keystore_backup.xml
apps/dk.brunnerne.onevoice/f/offboarding-checklist.txt
apps/dk.brunnerne.onevoice/f/logs/onevoice.log
apps/dk.brunnerne.onevoice/f/statement-archive.seal
apps/dk.brunnerne.onevoice/db/statements.db
apps/dk.brunnerne.onevoice/db/notes.db
```

Only one application is present:

```text
dk.brunnerne.onevoice
```

Its manifest shows that backups are allowed:

```text
"allowBackup": true
```

It also uses a custom:

```text
StatementBackupAgent
```

This is interesting because the backup contains application-private data such as:

* `sp/` → SharedPreferences
* `db/` → SQLite databases
* `f/` → Application files

So the backup gives us much more than just ordinary user-visible data.

---

# 3. Finding the Encryption Key

The most interesting file is:

```text
apps/dk.brunnerne.onevoice/sp/keystore_backup.xml
```

Inspecting it gives:

```xml
<!--
  Written by StatementStore on first run (CAKE-518).

  The content key is generated on the device and never leaves it. It is
  kept here so that clearing the app's cache does not lose the sealed
  statement archive, which support had to re-download several times
  during the pilot.
-->
<map>
    <string name="content_key">6mqMbv76RDT1G7yib5XrsS5DolJ+pPfZAGacZN3cTsc=</string>
    <string name="content_key_alg">AES-256-GCM</string>
    <string name="sealed_layout">nonce[12] || ciphertext || tag[16]</string>
    <string name="sealed_aad">none</string>
    <int name="content_key_version" value="3" />
    <boolean name="device_bound" value="true" />
</map>
```

This gives us everything required for decryption.

### Encryption details

| Property           | Value          |   |            |   |      |
| ------------------ | -------------- | - | ---------- | - | ---- |
| Algorithm          | AES-256-GCM    |   |            |   |      |
| Key                | Base64 encoded |   |            |   |      |
| Nonce              | 12 bytes       |   |            |   |      |
| Authentication tag | 16 bytes       |   |            |   |      |
| AAD                | None           |   |            |   |      |
| Layout             | `nonce         |   | ciphertext |   | tag` |

The key is:

```text
6mqMbv76RDT1G7yib5XrsS5DolJ+pPfZAGacZN3cTsc=
```

The important security mistake is that the encryption key was stored in:

```text
SharedPreferences
```

instead of Android Keystore.

Therefore, `allowBackup` caused the key to be exported alongside the encrypted data.

In other words:

```text
Encrypted database
        +
Encryption key
        ↓
Same Android backup
        ↓
Offline attacker
        ↓
Decrypt everything
```

The supposed `"device_bound": true` protection is therefore meaningless once the key itself leaves the device.

---

# 4. Decrypting `notes.db`

The database:

```text
db/notes.db
```

contains a `messages` table containing encrypted message blobs.

The messages are stored as Base64-encoded AES-GCM ciphertext.

The `peers` table contains the corresponding display names.

We can decrypt the messages using Python:

```python
import base64
import sqlite3
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

key = base64.b64decode(
    "6mqMbv76RDT1G7yib5XrsS5DolJ+pPfZAGacZN3cTsc="
)

aesgcm = AESGCM(key)


def decrypt(sealed_b64):
    raw = base64.b64decode(sealed_b64)

    # nonce[12] || ciphertext || tag[16]
    nonce = raw[:12]
    ciphertext_and_tag = raw[12:]

    return aesgcm.decrypt(
        nonce,
        ciphertext_and_tag,
        None
    ).decode()


con = sqlite3.connect("db/notes.db")
cur = con.cursor()

cur.execute("""
    SELECT
        p.display,
        m.direction,
        m.sent_ms,
        m.sealed
    FROM messages m
    JOIN peers p
        ON p.peer_id = m.peer_id
    ORDER BY m.sent_ms
""")

for display, direction, sent_ms, sealed in cur.fetchall():
    message = decrypt(sealed)

    print(
        f"[{sent_ms}] "
        f"{display} ({direction}): "
        f"{message}"
    )
```

Running the script decrypts all 12 messages.

The final messages are particularly interesting:

```text
Mette Holm (in):
One more thing before you go. The offboarding checklist
says IT images the handset before the wipe. Ask them
where the image goes.

Mette Holm (out):
Already did. It goes on the shared drive, with everything
on it. brunner{4llowB4ckup_t00k_th3_k3y_t00}

Mette Holm (in):
Of course it does.
```

And there is our flag.

---

# Flag

```text
brunner{4llowB4ckup_t00k_th3_k3y_t00}
```

---

# Root Cause

The vulnerability is essentially a **backup/key-management failure**.

### 1. Backups were enabled

The application used:

```text
android:allowBackup="true"
```

and a custom backup agent.

This allowed sensitive application data to be included in the Android backup.

### 2. The encryption key was stored with the application data

The AES-256-GCM key was stored inside:

```text
SharedPreferences
```

instead of Android Keystore.

Therefore, the backup contained both:

```text
Encrypted data
+
Encryption key
```

### 3. The backup itself was not encrypted

The `.ab` file was explicitly:

```text
Compressed, Not-Encrypted
```

So once an attacker obtained the backup, they could extract the application's private files without needing the original device.

### 4. "Device-bound" wasn't actually device-bound

The metadata claimed:

```xml
<boolean name="device_bound" value="true" />
```

But a key that can be exported with the application backup is not truly protected by the device.

---

# Fix

A secure implementation should address both **backup policy** and **key storage**.

### Disable unnecessary backups

If application data should never leave the device:

```xml
<application
    android:allowBackup="false">
```

Alternatively, use an appropriate backup configuration to explicitly exclude sensitive files.

### Use Android Keystore

Encryption keys should be stored in:

```text
Android Keystore
```

rather than:

```text
SharedPreferences
```

The Keystore can provide non-exportable, device-protected key material.

### Protect phone images

A phone image can contain:

* application databases
* credentials
* tokens
* encryption keys
* private messages
* personal information

Therefore, imaging artifacts should be treated as sensitive secrets.

The challenge also hints that the organization's offboarding policy was incomplete: the image was retained on a shared drive even though it contained the very secrets needed to decrypt the application data.

---

# Takeaway

The interesting lesson isn't that AES-256-GCM was broken.

It wasn't.

The cryptography was perfectly fine.

The failure was **key management**.

The application effectively did this:

```text
AES-256-GCM
      ↓
Strong encryption
      ↓
Key stored in SharedPreferences
      ↓
allowBackup = true
      ↓
Key gets backed up
      ↓
Ciphertext gets backed up
      ↓
Encryption becomes useless
```

So the real vulnerability was:

> **Strong encryption cannot protect data when the encryption key is exported together with the encrypted data.**

**Flag:** `brunner{4llowB4ckup_t00k_th3_k3y_t00}`
