# Slis

**Category:** Crypto
**Difficulty:** Easy-Medium
**Author:** Bond

## Challenge

```python
flag = 'brunner{' + input() + '}'
n = int.from_bytes(flag.encode())
lis = [n//(i+2) - n//(i+3) for i in range(9**5)]
assert sum(lis) == 22263691028918788395010325066307464924652601045336492930678310479674861811846
```

### Hint

> Brunnerne likes food 😋 and: Some like it sHOrT 🍲 So we'll keep it short ☺️

---

## Solution

### 1. Identify the telescoping sum

The list contains:

```python
n//(i+2) - n//(i+3)
```

for:

```python
i = 0 ... 9**5 - 1
```

Let:

```text
N = 9⁵ = 59049
```

The sum becomes:

```text
Σ [n//(i+2) - n//(i+3)]
```

Writing out the first few terms:

```text
(n//2 - n//3)
+ (n//3 - n//4)
+ (n//4 - n//5)
+ ...
+ (n//59050 - n//59051)
```

Everything in the middle cancels.

Therefore:

```text
Σ = n//2 - n//59051
```

So the assertion gives us:

```text
n//2 - n//59051 = S
```

where:

```python
S = 22263691028918788395010325066307464924652601045336492930678310479674861811846
```

---

### 2. Recover `n`

We need to find an integer `n` satisfying:

```python
f(n) = n//2 - n//59051 = S
```

Although `n//59051` is relatively small compared with `n//2`, we should solve the equation exactly.

The function:

```python
f(n) = n//2 - n//59051
```

is monotonic, so binary search can be used to find the smallest and largest `n` for which:

```python
f(n) == S
```

First, find the smallest possible `n`:

```python
def f(n):
    return n // 2 - n // 59051

lo, hi = 1, 1 << 400

while lo < hi:
    mid = (lo + hi) // 2

    if f(mid) < S:
        lo = mid + 1
    else:
        hi = mid

n0 = lo
```

Then find the largest possible `n`:

```python
lo, hi = n0, 1 << 400

while lo < hi:
    mid = (lo + hi + 1) // 2

    if f(mid) > S:
        hi = mid - 1
    else:
        lo = mid

n1 = hi
```

This leaves only a very small number of candidate values.

---

### 3. Convert `n` back to bytes

The original challenge created `n` using:

```python
n = int.from_bytes(flag.encode())
```

Python's default byte order here is **big-endian**.

Therefore, we can recover the original bytes using:

```python
n.to_bytes((n.bit_length() + 7) // 8, 'big')
```

Since we know the flag format is:

```text
brunner{...}
```

we can check whether the recovered bytes start with `brunner{` and end with `}`.

---

## Full Solve Script

```python
S = 22263691028918788395010325066307464924652601045336492930678310479674861811846

N = 9**5

def f(n):
    return n // 2 - n // (N + 2)


# Find the smallest n such that f(n) >= S
lo, hi = 1, 1 << 400

while lo < hi:
    mid = (lo + hi) // 2

    if f(mid) < S:
        lo = mid + 1
    else:
        hi = mid

n0 = lo


# Find the largest n such that f(n) <= S
lo, hi = n0, 1 << 400

while lo < hi:
    mid = (lo + hi + 1) // 2

    if f(mid) > S:
        hi = mid - 1
    else:
        lo = mid

n1 = hi


# Test all candidates
for n in range(n0, n1 + 1):
    nbytes = (n.bit_length() + 7) // 8
    data = n.to_bytes(nbytes, 'big')

    if data.startswith(b'brunner{') and data.endswith(b'}'):
        print(data.decode())
```

### Output

```text
brunner{Pease_porridge_sHOrT_:)}
```

---

## Flag

```text
brunner{Pease_porridge_sHOrT_:)}
```

## Key Takeaway

The main trick was recognizing the **telescoping sum**:

```text
(n//2 - n//3)
+ (n//3 - n//4)
+ ...
+ (n//59050 - n//59051)
```

which collapses to:

```text
n//2 - n//59051
```

From there, binary search recovers the possible value of `n`, and converting the integer back to bytes reveals the flag.

The flag also matches the food-related hint: **“Pease porridge hot, pease porridge cold...”**, with the challenge emphasizing **sHOrT**. 😋🍲
