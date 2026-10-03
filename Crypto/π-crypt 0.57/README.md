# π-crypt 0.57

**Category:** Crypto
**Difficulty:** Hard
**Points:** 100
**Flag:** `brunner{NB!:_Re-using_the_same_key_without_salting-and-hashing_risks_security_of_all_use-instances,_if_one_instance_leaks_info!}`

---

## Challenge Files

* `bake.py` — Encryption/decryption script
* `unbaked_pi.txt` — 1000 digits of π used as a keystream table
* `baked_pie.txt` — 256-character ciphertext

---

## Challenge Overview

The challenge encrypts the flag using two custom layers:

1. `pie_crypt` — a π-based Vigenère-like stream cipher.
2. `custom_ingredient` — a custom 16-round construction described as a "Feistel network".

At first glance, the construction looks complicated because of the π table and 16 rounds of mixing.

However, the important observation is that the entire second layer is **linear modulo 100** and operates independently on every character position.

That makes the construction reducible to a small system of linear equations.

---

# 1. Understanding `pie_crypt`

The first layer is:

```python
def pie_crypt(text: str, key: str, decrypt: bool = False) -> str:
    out = ""
    i = sum(base.index(c) for c in key)
    j = 0

    for c in text:
        d1 = int(pie[i % len(pie)])
        i += base.index(key[j % len(key)])
        j += 1

        d2 = int(pie[i % len(pie)])
        i += base.index(key[j % len(key)])
        j += 1

        shift = 10 * d1 + d2

        out += base[
            (base.index(c) + (-shift if decrypt else shift)) % len(base)
        ]

    return out
```

The important details are:

* `base` contains exactly 100 characters.
* Every character therefore maps to an integer in `[0, 99]`.
* Each plaintext character receives a shift between `00` and `99`.
* The shift is generated from digits of π.
* The key determines how the pointer moves through the π table.

So `pie_crypt` itself is essentially a custom Vigenère-style stream cipher over `Z/100Z`.

The difficult part is recovering the key.

---

# 2. Understanding `custom_ingredient`

The second layer uses:

```python
def custom_xor(s1, s2, decrypt=False):
    return "".join(
        base[
            (base.index(c1)
             + base.index(c2) * (-1 if decrypt else 1))
            % len(base)
        ]
        for c1, c2 in zip(s1, s2)
    )
```

Despite being called `custom_xor`, this is **not XOR**.

It is simply modular addition:

$$
a \oplus b = (a+b)\pmod{100}
$$

The round function is:

```python
def round_function(previous_left, previous_right, left, right):
    new_left  = previous_right
    new_right = custom_xor(
        previous_left,
        custom_xor(previous_right, key)
    )

    return left, right, new_left, new_right
```

If we represent the four 64-character blocks as numeric values:

$$
(p,q,r,s)
$$

then one round becomes:

$$
(p,q,r,s)
\rightarrow
(r,s,q,p+q+k)
\pmod{100}
$$

where `k` is the key character value at that position.

So:

```text
T(p,q,r,s,k) = (r, s, q, p + q + k) mod 100
```

---

# 3. The Important Vulnerability

There are two critical weaknesses.

## 3.1 No diffusion between character positions

Every character position is processed independently.

For example, position `0` never interacts with position `1`.

Therefore the 256-character ciphertext is actually **64 independent problems**.

Each position has only three unknown values:

* `x` — first half of the `pie_crypt` output
* `y` — second half of the `pie_crypt` output
* `k` — key character at that position

---

## 3.2 The construction is completely linear

The round operation only performs:

* copying
* addition
* modulo 100

There is no non-linearity.

The initial state is:

```text
(0, 0, x, y)
```

because the two seed blocks consist entirely of `'A'`.

Since:

```python
base.index('A') == 0
```

the seed contributes zero.

Therefore, after 16 rounds, every output value is simply a linear combination of:

$$
x,\ y,\ k
$$

---

# 4. Recovering the Linear Coefficients

Instead of manually expanding 16 rounds, we can simulate the transformation using symbolic unit inputs.

```python
def T(state, k):
    p, q, r, s = state
    return (r, s, q, (p + q + k) % 100)


def simulate(x0, y0, k0, rounds=16):
    state = (0, 0, x0, y0)

    for _ in range(rounds):
        state = T(state, k0)

    return state


A = simulate(1, 0, 0)
B = simulate(0, 1, 0)
C = simulate(0, 0, 1)
```

This gives the coefficients of `x`, `y`, and `k`.

After 16 rounds:

$$
\begin{aligned}
P_{16} &= 33k \pmod{100}\\
Q_{16} &= 54k \pmod{100}\\
R_{16} &= 13x+21y+33k \pmod{100}\\
S_{16} &= 21x+34y+54k \pmod{100}
\end{aligned}
$$

This is the key observation.

---

# 5. Recovering the Key

The first ciphertext block gives:

$$
P_{16}=33k\pmod{100}
$$

Since:

$$
\gcd(33,100)=1
$$

33 has a modular inverse modulo 100.

Therefore:

$$
k=33^{-1}P_{16}\pmod{100}
$$

So every key character can be recovered directly.

We can verify the result using the second block:

$$
Q_{16}=54k\pmod{100}
$$

If the recovered `k` is correct, this equation must hold.

---

# 6. Recovering `x` and `y`

Once `k` is known, subtract its contribution:

$$
r'=R_{16}-33k
$$

$$
s'=S_{16}-54k
$$

Leaving:

$$
\begin{bmatrix}
13 & 21\\
21 & 34
\end{bmatrix}
\begin{bmatrix}
x\\
y
\end{bmatrix}
=
\begin{bmatrix}
r'\\
s'
\end{bmatrix}
\pmod{100}
$$

The determinant is:

$$
13(34)-21(21)
$$

$$
=442-441
$$

$$
=1
$$

Since:

$$
1^{-1}\equiv1\pmod{100}
$$

the matrix is trivially invertible.

Its inverse is:

$$
\begin{bmatrix}
34 & -21\\
-21 & 13
\end{bmatrix}
\pmod{100}
$$

Therefore:

$$
x=34r'-21s'\pmod{100}
$$

$$
y=-21r'+13s'\pmod{100}
$$

---

# 7. Full Key-Recovery Script

The following script recovers the key directly from `baked_pie.txt`.

```python
base = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789æøåÆØÅ .,!?-:()[]/{}=<>+_@^|~%$#&*`'';"
assert len(base) == 100

with open("baked_pie.txt", encoding="utf-8") as f:
    ct = f.read().rstrip("\n")

n = len(ct) // 4

blocks = [
    ct[i * n:(i + 1) * n]
    for i in range(4)
]

P16, Q16, R16, S16 = [
    [base.index(c) for c in block]
    for block in blocks
]


def egcd(a, b):
    if a == 0:
        return b, 0, 1

    g, x1, y1 = egcd(b % a, a)

    return (
        g,
        y1 - (b // a) * x1,
        x1
    )


def modinv(a, m):
    g, x, _ = egcd(a, m)

    assert g == 1

    return x % m


inv33 = modinv(33, 100)

det = (13 * 34 - 21 * 21) % 100
invdet = modinv(det, 100)


def solve_xy(r, s, k):
    r_adj = (r - 33 * k) % 100
    s_adj = (s - 54 * k) % 100

    x = (
        invdet *
        (34 * r_adj - 21 * s_adj)
    ) % 100

    y = (
        invdet *
        (-21 * r_adj + 13 * s_adj)
    ) % 100

    return x, y


key_idx = []

for i in range(64):
    p = P16[i]
    q = Q16[i]
    r = R16[i]
    s = S16[i]

    # Recover k from P16 = 33k mod 100
    k = (p * inv33) % 100

    # Verify using Q16 = 54k mod 100
    assert (54 * k) % 100 == q

    key_idx.append(k)


key = "".join(base[k] for k in key_idx)

print("Recovered key:")
print(key)
```

---

# 8. Recovered Key

The script recovers the key directly:

```text
A_key_can_be_strong_just_by_being_long..._sometimes_at_least_...
```

No brute force is required.

No guessing is required.

The key is obtained algebraically from the ciphertext.

---

# 9. Decrypting the Flag

Instead of manually implementing the inverse of `pie_crypt`, we can reuse the original challenge's decryption routine.

Create a `secret.py` satisfying the assertions in `bake.py`:

```python
key = "A_key_can_be_strong_just_by_being_long..._sometimes_at_least_..."

flag = "brunner{x}"
```

Then:

```python
import bake

bake.key = "A_key_can_be_strong_just_by_being_long..._sometimes_at_least_..."

bake.main(decrypt=True)
```

This produces:

```text
brunner{NB!:_Re-using_the_same_key_without_salting-and-hashing_risks_security_of_all_use-instances,_if_one_instance_leaks_info!}
```

---

# 10. Flag

```text
brunner{NB!:_Re-using_the_same_key_without_salting-and-hashing_risks_security_of_all_use-instances,_if_one_instance_leaks_info!}
```

---

# 11. Why the Scheme Breaks

The "Feistel network" looks complicated because it performs 16 rounds, but the rounds do not provide real cryptographic security.

The transformation is:

$$
(p,q,r,s)\rightarrow(r,s,q,p+q+k)
\pmod{100}
$$

This is linear.

Additionally:

* every character position is independent;
* the initial state is zero;
* the same key is reused;
* there is no nonce or salt;
* `custom_xor` is modular addition rather than XOR;
* there is no non-linear operation;
* 16 rounds do not introduce meaningful cryptographic complexity.

Thus the entire construction collapses into four linear equations.

---

# 12. Takeaway

The challenge demonstrates an important cryptographic lesson:

> **More rounds and complicated-looking transformations do not automatically make a cipher secure.**

A secure construction needs properties such as:

* proper key derivation;
* unique nonces/IVs;
* non-linear mixing;
* diffusion between positions;
* resistance to known/chosen-plaintext attacks;
* cryptographically reviewed primitives.

Here, the combination of an all-zero initial state and a linear modular recurrence makes the entire construction solvable with elementary modular arithmetic.

The flag itself summarizes the core lesson:

```text
Re-using the same key without salting and hashing risks security
of all use-instances if one instance leaks information.
```

---

## Exploit Summary

```text
256-char ciphertext
        │
        ▼
Split into 4 × 64 blocks
        │
        ▼
Model 16 rounds as linear recurrence
        │
        ▼
P16 = 33k (mod 100)
        │
        ▼
Recover k using 33⁻¹
        │
        ▼
Verify using Q16 = 54k
        │
        ▼
Solve 2×2 system for x,y
        │
        ▼
Recovered pie_crypt key
        │
        ▼
Run original decrypt routine
        │
        ▼
        🚩 FLAG
```

**Core vulnerability:** Linear modular arithmetic + zero seed + independent character positions + key reuse.
