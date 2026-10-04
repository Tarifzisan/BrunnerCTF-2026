# Free Play — Forensics Writeup

**Category:** Forensics
**Difficulty:** Easy-Medium
**Flag:** `brunner{strong_force_in_you}`

---

## Challenge Description

IT found a **LEGO Star Wars: The Complete Saga (2009)** save file on a Procurement employee's corporate laptop.

HR wants to know why he was **"staring at his character roster" for three weeks.**

### Handout

* `SaveGame1` — 40,572-byte binary save file
* `Game.jpg` — Screenshot of the character-select screen showing a mysterious custom character behind a `?` icon

---

## 1. Initial Triage

First, I checked the file type and inspected the beginning of the binary data:

```bash
file SaveGame1
od -A x -t x1z SaveGame1 | head
```

The hex dump started with:

```text
000000 48 4d 47 52 01 00 00 00 28 20 00 00 ...   >HMGR............<
```

The `HMGR` magic value, together with readable UTF-16LE strings such as:

```text
LEGO Star Wars
LucasArts
GAME.DAT
Episode_I.DAT
```

confirmed that this was a standard **LEGO Star Wars: The Complete Saga save slot**.

There was no obvious custom container or archive format.

---

## 2. Searching for Suspicious Data

Next, I extracted printable ASCII strings along with their offsets:

```python
import re

data = open('SaveGame1', 'rb').read()

for m in re.finditer(rb'[\x20-\x7e]{4,}', data):
    print(m.start(), m.group())
```

Most of the results looked completely normal for a game save, including input glyph names:

```text
[CROSS]
[JUMP]
```

and sound-related strings such as:

```text
wpn_bib_stab
```

However, two unusual strings appeared very close to the end of the file:

```text
40032  STRANGER 1
40088  STRANGER 2
```

These names correspond to **custom-built LEGO minifigures** that can be created and saved in the game.

That was particularly interesting because the provided screenshot showed a mysterious custom character in the character roster.

---

## 3. Inspecting the Custom Character Record

I then examined the bytes surrounding the `STRANGER 2` record.

```python
data = open('SaveGame1', 'rb').read()

seg = data[40224:40400]

print(list(seg))
```

The output contained a long sequence consisting almost entirely of two values:

```text
[0, 3, 3, 3, 0, 0, 3, 3, 0, 3, 3, 3, 0, 3, 0, 0,
 0, 3, 3, 3, 0, 0, 3, 0, ...]
```

The important observation was that the field only used:

```text
0x00
0x03
```

This strongly suggested that the values were being used as a **binary bit channel**:

```text
0x00 → 0
0x03 → 1
```

After removing the trailing zero-padding, there were exactly **152 meaningful entries**.

And:

```text
152 = 19 × 8
```

So the data contained exactly **19 bytes worth of bits**, which was a strong indication that it could represent ASCII text.

---

## 4. Decoding the Bitstream

I mapped the two byte values to binary bits:

```text
0x00 → 0
0x03 → 1
```

Then grouped the resulting bitstream into groups of 8 and converted each group into an ASCII character.

```python
seg = data[40224:40376]

bits = ''.join(
    '1' if b == 3 else '0'
    for b in seg
)

text = ''.join(
    chr(int(bits[i:i+8], 2))
    for i in range(0, len(bits), 8)
)

print(text)
```

The result was:

```text
strong_force_in_you
```

So the seemingly ordinary custom-character data was actually being used as a **covert bit channel**.

Each array entry represented one bit:

```text
0x00 → 0
0x03 → 1
```

Those bits reconstructed the hidden message.

---

## 5. Flag

The decoded message was:

```text
strong_force_in_you
```

Therefore, the flag is:

```text
brunner{strong_force_in_you}
```

---

## 6. Takeaway

This challenge demonstrates how seemingly harmless application data can be abused as a **covert storage channel**.

The custom LEGO character was essentially a distraction. The important clue was the unusual sequence of only two byte values following the character data.

The key observations were:

1. `HMGR` identified the file as a LEGO Star Wars save.
2. `STRANGER 1` and `STRANGER 2` indicated custom characters.
3. The character data contained a suspicious sequence using only `0x00` and `0x03`.
4. Mapping those values to `0` and `1` produced a 152-bit stream.
5. Grouping the bits into bytes revealed the hidden message.
6. The message gave the flag.

### Final Flag

```text
brunner{strong_force_in_you}
```
