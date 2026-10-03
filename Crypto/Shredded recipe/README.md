# Shredded Recipe

**Category:** Crypto
**Difficulty:** Hard

## Challenge Description

We are given the following `source.py`:

```python
from Crypto.Util.number import getPrime, bytes_to_long
import random

flag = b'brunner{?????????????????????????????????????????????}'
assert len(flag) == 54

p = getPrime(512)
print(p)

x = bytes_to_long(flag[0::3])
y = bytes_to_long(flag[1::3])
z = bytes_to_long(flag[2::3])

a, b, c = (random.randint(2, p) for _ in range(3))
d = (a*x + b*y + c*z) % p
print(a, b, c, d)
```

The flag is 54 bytes long and is split into three interleaved parts:

```text
x = flag[0::3]
y = flag[1::3]
z = flag[2::3]
```

Each part contains 18 bytes, giving three relatively small integers `x`, `y`, and `z`.

The challenge then publishes:

```text
a*x + b*y + c*z ≡ d (mod p)
```

where `p` is a 512-bit prime.

At first this looks underdetermined because we have one equation and three unknowns. However, the important observation is that `x`, `y`, and `z` are only 18 bytes each, while the modulus is 512 bits.

This makes the problem suitable for a **lattice-based bounded modular equation attack**.

---

## 1. Exploiting the Flag Format

The flag has the form:

```text
brunner{.............................................}
```

and has exactly 54 bytes.

The first eight bytes are known:

```text
brunner{
```

and the final byte is known:

```text
}
```

Because the flag is split every three bytes, the known bytes are distributed among the three streams.

| Stream           | Flag indices     | Known bytes        | Unknown bytes |
| ---------------- | ---------------- | ------------------ | ------------- |
| `x = flag[0::3]` | 0, 3, 6, ..., 51 | positions 0, 3, 6  | 15 bytes      |
| `y = flag[1::3]` | 1, 4, 7, ..., 52 | positions 1, 4, 7  | 15 bytes      |
| `z = flag[2::3]` | 2, 5, 8, ..., 53 | positions 2, 5, 53 | 15 bytes      |

Therefore, each 18-byte integer can be represented as a known part plus an unknown part.

We write:

```text
x = Kx + Vx
y = Ky + Vy
z = Kz + 256*Vz
```

The `256` factor in `z` appears because the final byte of `z` is the known `}` byte. Therefore, the unknown portion occupies the higher 15 bytes of `z`.

Each unknown value satisfies approximately:

```text
0 <= Vx, Vy, Vz < 2^120
```

---

## 2. Reducing the Equation

Substituting the above representation into:

```text
a*x + b*y + c*z ≡ d (mod p)
```

gives:

```text
a(Kx + Vx) + b(Ky + Vy) + c(Kz + 256Vz) ≡ d (mod p)
```

Collecting the unknown terms:

```text
a*Vx + b*Vy + 256*c*Vz
    ≡ d - a*Kx - b*Ky - c*Kz (mod p)
```

Define:

```text
c' = 256*c mod p
```

and:

```text
D = (d - a*Kx - b*Ky - c*Kz) mod p
```

Then the problem becomes:

```text
a*Vx + b*Vy + c'*Vz ≡ D (mod p)
```

with:

```text
Vx, Vy, Vz < 2^120
```

This is the key reduction.

---

## 3. Lattice Construction

We construct the following lattice basis:

```text
R0 = (p, 0, 0, 0)
R1 = (a, 1, 0, 0)
R2 = (b, 0, 1, 0)
R3 = (c', 0, 0, 1)
```

So the basis matrix is:

```text
[ p  0  0  0 ]
[ a  1  0  0 ]
[ b  0  1  0 ]
[ c' 0  0  1 ]
```

A general lattice vector is:

```text
k0*R0 + Vx*R1 + Vy*R2 + Vz*R3
```

which equals:

```text
(k0*p + a*Vx + b*Vy + c'*Vz,
 Vx,
 Vy,
 Vz)
```

Because:

```text
a*Vx + b*Vy + c'*Vz ≡ D (mod p)
```

there exists some integer `k0` such that:

```text
k0*p + a*Vx + b*Vy + c'*Vz = D
```

Therefore, the corresponding lattice vector is:

```text
(D, Vx, Vy, Vz)
```

Our target vector is:

```text
t = (D, 0, 0, 0)
```

The difference between the target and the solution lattice vector is:

```text
(0, Vx, Vy, Vz)
```

Since each unknown is only around 120 bits, this difference is relatively small compared with the 512-bit modulus.

This allows us to formulate the problem as a **Closest Vector Problem (CVP)**.

---

## 4. Solving with LLL + CVP

We can use `fpylll` to reduce the lattice and solve the CVP.

```python
from fpylll import IntegerMatrix, LLL, CVP

rows = [
    [p,  0, 0, 0],
    [a,  1, 0, 0],
    [b,  0, 1, 0],
    [cp, 0, 0, 1],
]

B = IntegerMatrix(4, 4)

for i in range(4):
    for j in range(4):
        B[i, j] = rows[i][j]

LLL.reduction(B)

v = CVP.closest_vector(B, (D, 0, 0, 0))

Vx = v[1]
Vy = v[2]
Vz = v[3]
```

The recovered values have the expected size and satisfy the original modular equation.

---

## 5. Reconstructing the Three Streams

After recovering `Vx`, `Vy`, and `Vz`:

```python
Ux = Vx
Uy = Vy
Uz = 256 * Vz

x = Kx + Ux
y = Ky + Uy
z = Kz + Uz
```

Convert them back into their original 18-byte representations:

```python
xb = x.to_bytes(18, 'big')
yb = y.to_bytes(18, 'big')
zb = z.to_bytes(18, 'big')
```

Now interleave the three streams:

```python
flag = bytearray(54)

flag[0::3] = xb
flag[1::3] = yb
flag[2::3] = zb

print(flag)
```

This reconstructs the original 54-byte flag.

---

## 6. Verification

Finally, verify that the recovered values satisfy the original equation:

```python
assert (a*x + b*y + c*z) % p == d
```

The assertion succeeds.

The reconstructed flag is:

```text
brunner{i_really_love_solving_equations_with_lattices}
```

## Flag

```text
brunner{i_really_love_solving_equations_with_lattices}
```

---

## 7. Full Exploit Script

A complete solver can be written as follows:

```python
from Crypto.Util.number import bytes_to_long
from fpylll import IntegerMatrix, LLL, CVP


# Read challenge output
with open("output.txt") as f:
    p = int(f.readline().strip())
    a, b, c, d = map(int, f.readline().split())


# Known flag bytes
prefix = b"brunner{"
suffix = b"}"


# Build the known parts of each interleaved stream.
#
# Unknown bytes are represented by zeroes.
known
```
