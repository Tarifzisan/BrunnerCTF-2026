# Magic or not — Writeup

**Category:** Misc
**Difficulty:** Easy
**Points:** 100
**Author:** H4N5

## Challenge Description

> As an intern at **Brunner Corporation**, I developed a **cutting-edge** image obfuscation algorithm, designed to hide sensitive image data.
>
> I'm confident it's secure, but the security team isn't convinced. They believe custom cryptography always hides a flaw.
>
> Can you prove them wrong? Analyze the implementation, recover the original image, and find the flag.

---

## Files

The challenge provided four files:

```text
Brunner1.jpg
Brunner2.gif
Brunner3.png
Brunner4.bmp
```

Although they had image extensions, none of them were recognized as valid image files.

---

## 1. Checking the File Types

First, I checked the files with `file`:

```bash
file Brunner*
```

Output:

```text
Brunner1.jpg: data
Brunner2.gif: data
Brunner3.png: data
Brunner4.bmp: data
```

So the extensions were misleading.

I also checked the first few bytes:

```bash
for f in Brunner*; do
    echo "===== $f ====="
    xxd -l 32 "$f"
done
```

The beginning of the files looked like:

```text
Brunner1.jpg:
a7 80 a7 b3 7b 42 12 08 41 58 58 58 58 59 58 58 ...

Brunner2.gif:
10 1e 11 6f 6e 36 e9 57 d7 54 a0 57 57 57 57 57 ...

Brunner3.png:
d0 09 17 1e 54 53 43 53 59 59 59 54 10 11 1d 0b ...

Brunner4.bmp:
18 17 d0 d4 4b 5a 5a 5a 5a 5a d0 5a 5a 5a ...
```

There was a very noticeable pattern: each file contained a large number of the same byte.

---

## 2. Identifying the XOR Keys

The repeated bytes were:

| File           | Repeated byte |    Hex |
| -------------- | ------------: | -----: |
| `Brunner1.jpg` |           `X` | `0x58` |
| `Brunner2.gif` |           `W` | `0x57` |
| `Brunner3.png` |           `Y` | `0x59` |
| `Brunner4.bmp` |           `Z` | `0x5A` |

This suggested a simple XOR-based obfuscation.

For example, a JPEG normally starts with:

```text
FF D8 FF
```

For `Brunner1.jpg`:

```text
A7 80 A7
```

XORing with `0x58` gives:

```text
A7 ^ 58 = FF
80 ^ 58 = D8
A7 ^ 58 = FF
```

So:

```text
A7 80 A7
   XOR
58 58 58
   =
FF D8 FF
```

That's a valid JPEG magic header.

Therefore, each file was XORed with a single-byte key.

---

## 3. Decrypting the Files

I used the following Python script:

```python
from pathlib import Path

keys = {
    "Brunner1.jpg": 0x58,
    "Brunner2.gif": 0x57,
    "Brunner3.png": 0x59,
    "Brunner4.bmp": 0x5a,
}

for name, key in keys.items():
    data = Path(name).read_bytes()

    # Reverse the XOR obfuscation
    out = bytes(b ^ key for b in data)

    output = Path("decrypted_" + name)
    output.write_bytes(out)

    print(f"{name}: XOR 0x{key:02x} -> {output}")
```

Run:

```bash
python3 decrypt.py
```

Output:

```text
Brunner1.jpg: XOR 0x58 -> decrypted_Brunner1.jpg
Brunner2.gif: XOR 0x57 -> decrypted_Brunner2.gif
Brunner3.png: XOR 0x59 -> decrypted_Brunner3.png
Brunner4.bmp: XOR 0x5a -> decrypted_Brunner4.bmp
```

---

## 4. Confirming the Recovered Files

Now I checked the decrypted files:

```bash
file decrypted_*
```

Output:

```text
decrypted_Brunner1.jpg: JPEG image data, Exif Standard: [TIFF image data, big-endian, direntries=3], baseline, precision 8, 642x896, components 3

decrypted_Brunner2.gif: GIF image data, version 89a, 190 x 896

decrypted_Brunner3.png: PNG image data, 68 x 896, 8-bit/color RGBA, non-interlaced

decrypted_Brunner4.bmp: PC bitmap, Windows 98/2000 and newer format, 321 x 896 x 32
```

The files were successfully restored.

---

## 5. Analyzing the Image Dimensions

The dimensions revealed another important clue:

```text
Brunner1 → 642 × 896
Brunner2 → 190 × 896
Brunner3 →  68 × 896
Brunner4 → 321 × 896
```

All four images had exactly the same height:

```text
896 pixels
```

This strongly suggested that they were **vertical slices of one larger image**.

Their combined width is:

```text
642 + 190 + 68 + 321 = 1221
```

So the original image was likely:

```text
1221 × 896
```

---

## 6. Reconstructing the Original Image

I combined the four recovered images horizontally:

```python
from PIL import Image

files = [
    "decrypted_Brunner1.jpg",
    "decrypted_Brunner2.gif",
    "decrypted_Brunner3.png",
    "decrypted_Brunner4.bmp",
]

images = [Image.open(f).convert("RGB") for f in files]

width = sum(img.width for img in images)
height = max(img.height for img in images)

combined = Image.new("RGB", (width, height))

x = 0

for img in images:
    combined.paste(img, (x, 0))
    x += img.width

combined.save("combined.png")

print(f"Created combined.png: {width}x{height}")
```

Run:

```bash
python3 combine.py
```

This produced:

```text
combined.png
```

with dimensions:

```text
1221 × 896
```

Opening the reconstructed image revealed the hidden flag.

---

## 7. Flag

The flag was:

```text
brunner{ctf2026}
```

## Final Flag

```text
brunner{ctf2026}
```

---

## Summary

The challenge used a simple custom XOR obfuscation.

The solve path was:

```text
Fake image extensions
        ↓
Invalid magic bytes
        ↓
Repeated byte patterns
        ↓
Identify single-byte XOR keys
        ↓
XOR each file
        ↓
Recover four valid images
        ↓
Notice identical height (896 px)
        ↓
Combine images horizontally
        ↓
Recover original image
        ↓
brunner{ctf2026}
```

### Key Takeaway

Custom cryptography doesn't have to be complicated to be vulnerable. In this challenge, the repeated bytes made the XOR key easy to identify, and the image dimensions revealed how the recovered files should be reconstructed.
