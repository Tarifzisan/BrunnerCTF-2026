# OneVoice — Android Reverse Engineering

**Category:** Mobile
**Difficulty:** Medium
**Author:** KyootyBella

## Challenge

Brunnerne A/S ships an internal Android application called `OneVoice`, which displays the week's corporate-approved external communication statement.

Rumors are circulating about a major announcement, and the goal is to determine whether Corporate has already drafted it.

**Files provided:**

```text
OneVoice.apk
```

---

## 1. APK Recon

First, extract the APK:

```bash
unzip OneVoice.apk -d onevoice
cd onevoice
```

The APK contains the usual Android structure:

```text
classes.dex
resources.arsc
res/
assets/
AndroidManifest.xml
...
```

The class and resource names are heavily minified/obfuscated.

Searching `classes.dex` for application-specific strings reveals the package:

```text
dk.brunnerne.onevoice
```

and three interesting classes:

```text
dk/brunnerne/onevoice/MainActivity
dk/brunnerne/onevoice/StatementStore
dk/brunnerne/onevoice/BuildConfig
```

`StatementStore` immediately stands out because it is not a typical Android framework class.

---

## 2. Decompiling `classes.dex`

Neither `jadx` nor `apktool` was available in the environment, so I used **androguard** to inspect the DEX directly.

Install it with:

```bash
pip install androguard
```

Using `AnalyzeDex()` and `class.get_source()`, the application classes can be reconstructed.

The important logic is inside:

```text
dk.brunnerne.onevoice.StatementStore
```

---

## 3. Analyzing `MainActivity`

The `MainActivity.onCreate()` implementation is relatively simple.

The important part is essentially:

```java
TextView(week).setText(
    getString(R.string.week_fmt, "2026-W27")
);

String statement = StatementStore.INSTANCE.render(this);

TextView(statement).setText(
    statement != null
        ? statement
        : getString(R.string.statement_unavailable)
);
```

The UI doesn't contain any interesting hidden logic.

Instead, it calls:

```java
StatementStore.INSTANCE.render(this)
```

Therefore, the next step is to investigate `StatementStore`.

---

# 4. Reversing `StatementStore`

Several important pieces of logic can be recovered.

### Hardcoded table

There is a 16-byte table:

```java
TABLE = {
    63, -95, 8, -44,
    98, -100, 23, -27,
    75, 122, -61, 46,
    -111, 86, -67, -16
};
```

Converting the signed Java bytes to unsigned values gives:

```python
TABLE = bytes([
    63, 161, 8, 212,
    98, 156, 23, 231,
    75, 122, 195, 46,
    145, 86, 189, 240
])
```

---

## 5. Deriving the Salt

The application constructs the salt from the week displayed in the UI:

```text
onevoice-2026-W27
```

In Python:

```python
salt = b"onevoice-2026-W27"
```

The keystream byte is generated using:

```java
TABLE[i % 16] ^ salt[i % salt.length]
```

So:

```python
def keystream_byte(i, salt):
    return TABLE[i % 16] ^ salt[i % len(salt)]
```

---

# 6. Understanding the Decode Function

The application doesn't use a standard encryption algorithm.

Instead, each byte is:

1. Rotated right by `(i % 7) + 1`
2. XORed with the generated keystream byte

The equivalent Python implementation is:

```python
def rotate_right8(v, n):
    v &= 0xff
    n &= 7

    if n == 0:
        return v

    return ((v << (8 - n)) | (v >> n)) & 0xff


def decode(enc, salt):
    return bytes(
        rotate_right8(b, (i % 7) + 1)
        ^ keystream_byte(i, salt)
        for i, b in enumerate(enc)
    )
```

At this point we have enough information to reproduce the application's decryption logic.

But we still need to find the encrypted resource.

---

# 7. Finding `R.raw.messaging`

The code references:

```text
R.raw.messaging
```

However, there is no obvious:

```text
res/raw/messaging
```

because the resource filename has also been minified.

The resource table can be resolved using **androguard's `ARSCParser`**.

The resource ID resolves as:

```python
resid = arsc.get_res_id_by_key(
    pkg,
    'raw',
    'messaging'
)
```

which gives:

```text
0x7f0f0000
```

Resolving the resource configuration:

```python
arsc.get_resolved_res_configs(resid)
```

reveals:

```text
res/kD.bin
```

The file is approximately:

```text
637 bytes
```

So `res/kD.bin` is the encrypted payload referenced by:

```text
R.raw.messaging
```

---

# 8. Reversing `unpack()`

The resource is not simply one encrypted string.

The `unpack()` function parses multiple records from the binary blob.

The format is:

```text
[1 byte record count]
[2 byte big-endian length]
[record bytes]
[2 byte big-endian length]
[record bytes]
...
```

The Python implementation:

```python
def unpack(data):
    count = data[0]
    idx = 1
    out = []

    for _ in range(count):
        length = (data[idx] << 8) | data[idx + 1]
        idx += 2

        out.append(data[idx:idx + length])
        idx += length

    return out
```

This is an important discovery.

The resource contains **multiple records**, not just one statement.

---

# 9. Full Decryption Script

Combining everything:

```python
TABLE = bytes([
    63, 161, 8, 212,
    98, 156, 23, 231,
    75, 122, 195, 46,
    145, 86, 189, 240
])

salt = b"onevoice-2026-W27"


def keystream_byte(i, salt):
    return TABLE[i % 16] ^ salt[i % len(salt)]


def rotate_right8(v, n):
    v &= 0xff
    n &= 7

    if n == 0:
        return v

    return ((v << (8 - n)) | (v >> n)) & 0xff


def decode(enc, salt):
    return bytes(
        rotate_right8(b, (i % 7) + 1)
        ^ keystream_byte(i, salt)
        for i, b in enumerate(enc)
    )


def unpack(data):
    count = data[0]
    idx = 1
    out = []

    for _ in range(count):
        length = (data[idx] << 8) | data[idx + 1]
        idx += 2

        out.append(data[idx:idx + length])
        idx += length

    return out


with open("res/kD.bin", "rb") as f:
    data = f.read()

records = unpack(data)

print(f"[+] Records found: {len(records)}")

for i, record in enumerate(records):
    print(f"\n--- Record {i} ---")
    print(decode(record, salt).decode())
```

Running the script reveals:

```text
[+] Records found: 2
```

This confirms that the resource contains two separate messages.

---

# 10. Record 0 — Approved Statement

The first record decrypts to:

```text
Approved wording — week 2026-W27

Brunnerne A/S is aligning its footprint to demand. No decisions have
been taken, and we will communicate with colleagues before we
communicate externally.

Cleared by Corporate Communications. Please use verbatim.
```

This is the message displayed by the Android application.

Why?

Because `render()` explicitly uses:

```java
records[0]
```

and never processes the remaining records.

---

# 11. Record 1 — Hidden Draft

The second record is much more interesting:

```text
DRAFT — NOT FOR CIRCULATION — week 2026-W27

The Aarhus site closes at the end of Q3. Thirty-one roles go; the
board signed it off on 12 June. Communications will not confirm a
number until consultation has formally opened, so the line for now
is that no decisions have been taken.

Do not send this to anyone outside Communications.

brunner{th3_dr4ft_sh1pp3d_w1th_th3_4ppr0v4l}
```

The application never displays this record, but it is still shipped inside the APK.

---

# 12. Flag

The flag is:

```text
brunner{th3_dr4ft_sh1pp3d_w1th_th3_4ppr0v4l}
```

---

## Flag Submission

```text
brunner{th3_dr4ft_sh1pp3d_w1th_th3_4ppr0v4l}
```

---

# 13. Key Takeaways

### 1. Don't stop at what the UI displays

The application only displayed the first record:

```java
records[0]
```

But the underlying resource contained multiple records.

Whenever reversing an application, it's worth checking whether parsers, lists, arrays, or databases contain data that the UI never exposes.

---

### 2. `RECORD_APPROVED = 0` was a useful clue

The presence of a constant representing the approved record index strongly suggests that other records may exist.

In this case:

```text
RECORD_APPROVED = 0
```

effectively tells us:

```text
record 0 = approved
record 1+ = potentially something else
```

That makes the implementation of `unpack()` particularly interesting.

---

### 3. Obfuscating names doesn't protect the data

The application had obfuscated class/resource names:

```text
StatementStore
        ↓
resource name
        ↓
res/kD.bin
```

But the actual decryption algorithm was still present inside the APK.

The following were all available to the reverse engineer:

* Encryption/decryption algorithm
* Hardcoded `TABLE`
* Salt generation
* Record structure
* Resource ID
* Resource payload
* Record selection logic

Therefore, the obfuscation only made analysis slightly more annoying; it did not provide meaningful protection.

---

### 4. The key material was recoverable

The salt was derived from a value already visible in the UI:

```text
2026-W27
```

which produced:

```text
onevoice-2026-W27
```

The other component was hardcoded directly in the application:

```text
TABLE
```

So anyone capable of reversing the APK could reproduce the entire decoding process.

---

## Conclusion

The intended statement was:

> No
