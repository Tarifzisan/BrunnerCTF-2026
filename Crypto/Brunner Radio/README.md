# Brunner Radio

**Category:** Crypto 
**Difficulty:** Medium
**Points:** 100
**Author:** Bond

## Challenge

> To readily be able to share our delicious news with brunsviger-fans even in situations when there is no internet connection available 🚫 we at Brunnerne Inc. are also working on a radio-broadcasting scheme 📡
>
> ... But I forget: Which frequency were we transmitting on? 📻

We are given two files:

* `chal.py` — the challenge generator
* `total_broadcast.txt` — the generated broadcast data

---

## Understanding the Generator

There are **9 secret transmissions**, and each transmission is exactly **36 bytes** long.

```python
assert len(transmissions) == 9
assert "brunner{" in "".join(transmissions)
bytelen_transmission = 36
assert all(len(t) == bytelen_transmission for t in transmissions)
```

Each character is converted into 8 bits:

```python
bits = [[int(b) for c in t for b in f"{ord(c):08b}"] for t in transmissions]
```

Therefore:

```text
36 bytes × 8 = 288 bits
```

So each transmission contains exactly **288 bits**.

The transmissions are assigned wavelengths from `1` to `9`:

```python
for i in range(len(transmissions)):
    wavelength = i + 1
```

For every bit position, the bit is added to every time slot whose index is divisible by its wavelength:

```python
k = wavelength
while k <= bit_repetition_period:
    smoab[j][k - 1] += bits[i][j]
    k += wavelength
```

In mathematical form:

```text
smoab[j][k] = Σ bits[i][j]
              where (i + 1) divides (k + 1)
```

So each row of `total_broadcast.txt` represents one bit position, and contains **100 aggregated values**.

---

## The Key Observation

We don't actually need all 100 columns.

The first **9 columns** are enough.

For `k = 1..9`, every divisor of `k` is also at most `9`.

Therefore, when processing `k` from `1` to `9`, every contribution except the current wavelength has already been recovered.

This allows us to solve the transmissions one by one.

### Example

For `k = 1`:

```text
smoab[0] = bit[1]
```

So:

```text
bit[1] = smoab[0]
```

For `k = 2`:

```text
smoab[1] = bit[1] + bit[2]
```

Since `bit[1]` is already known:

```text
bit[2] = smoab[1] - bit[1]
```

For `k = 4`:

```text
smoab[3] = bit[1] + bit[2] + bit[4]
```

Thus:

```text
bit[4] = smoab[3] - bit[1] - bit[2]
```

The same idea works all the way up to wavelength `9`.

---

## Divisor Equations

| `k` | Divisors   | Equation                 |
| --: | ---------- | ------------------------ |
|   1 | 1          | `S₁ = b₁`                |
|   2 | 1, 2       | `S₂ = b₁ + b₂`           |
|   3 | 1, 3       | `S₃ = b₁ + b₃`           |
|   4 | 1, 2, 4    | `S₄ = b₁ + b₂ + b₄`      |
|   5 | 1, 5       | `S₅ = b₁ + b₅`           |
|   6 | 1, 2, 3, 6 | `S₆ = b₁ + b₂ + b₃ + b₆` |
|   7 | 1, 7       | `S₇ = b₁ + b₇`           |
|   8 | 1, 2, 4, 8 | `S₈ = b₁ + b₂ + b₄ + b₈` |
|   9 | 1, 3, 9    | `S₉ = b₁ + b₃ + b₉`      |

Thus, for each bit position, we can recover:

```text
b₁ → b₂ → b₃ → ... → b₉
```

This is essentially a simple **triangular system** that can be solved from top to bottom.

---

## Solution Script

```python
with open("total_broadcast.txt") as f:
    lines = f.read().splitlines()

bit_length = 288

smoab = [[int(c) for c in line] for line in lines]

# Divisors of 1..9
divisors = {
    k: [d for d in range(1, 10) if k % d == 0]
    for k in range(1, 10)
}

# bits[transmission][bit_position]
bits = [[0] * bit_length for _ in range(9)]

for j in range(bit_length):
    known = {}

    for k in range(1, 10):
        # Start with the aggregated value
        s = smoab[j][k - 1]

        # Remove contributions from already recovered wavelengths
        for d in divisors[k]:
            if d != k:
                s -= known[d]

        # The remaining value is the current transmission's bit
        known[k] = s
        bits[k - 1][j] = s


# Convert the recovered bits back into characters
transmissions = []

for i in range(9):
    b = bits[i]

    chars = [
        chr(
            int(
                "".join(map(str, b[x * 8:(x + 1) * 8])),
                2
            )
        )
        for x in range(36)
    ]

    transmissions.append("".join(chars))


for t in transmissions:
    print(repr(t))
```

---

## Recovered Transmissions

Running the script gives:

```text
'hine bright like a diamond. Shine br'
' takes the shot - and it goes in!! T'
'while other can-openers just open th'
'd. But in my opinion the even better'
"'s going to be cloudy, but sunshine "
"cause I'm happyyy. Clap along if you"
'brummer{...brrru-uuuuu-uuum-mmmm...}'
'brunner{Brunsviger_is_in_the_air_<3}'
'othello{Sikke_dog_en_dejlig_kage_:D}'
```

Most of the recovered transmissions are decoys.

Interestingly, several of them resemble flags from other challenges:

```text
brummer{...}
brunner{...}
othello{...}
```

The actual `brunner{...}` flag is:

```text
brunner{Brunsviger_is_in_the_air_<3}
```

## Flag

```text
brunner{Brunsviger_is_in_the_air_<3}
```

## Takeaway

The main trick is recognizing that the broadcast is not encryption in the traditional sense. Each wavelength contributes its bit whenever the wavelength divides the time slot.

Because the first 9 time slots contain all divisors needed for wavelengths `1..9`, the system can be recovered sequentially.

The important observation is:

> **Use the first 9 columns and solve the divisor contributions from wavelength 1 through 9.**
