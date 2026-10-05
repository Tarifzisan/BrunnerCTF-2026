# KPWhy — Writeup

**Category:** Reverse Engineering
**Difficulty:** Easy-Medium
**Flag:** `brunner{y0ur_kp1s_ar3_n0t_l00king_gr8_buddy}`

---

## Challenge

We're given a binary called `kpiman`.

The program asks for an **employee ID** and calculates a productivity score. Apparently, exactly one input achieves **100% productivity**.

```text
$ ./kpiman

BrunnerCorp KPIman v3.1
Enter employee ID: AAAA
Analyzing synergy...
Productivity: 0%. Have you considered a career in cake tasting?
```

Our goal is to reverse engineer the binary and recover the valid 44-byte input.

---

## 1. Initial Triage

First, identify the binary:

```bash
file kpiman
```

Output:

```text
kpiman: ELF 64-bit LSB executable, x86-64, ... not stripped
```

The binary is **not stripped**, which is useful because function and variable names are still present.

Let's inspect the strings:

```bash
strings kpiman
```

Among the interesting strings/functions we find:

```text
calculateSynergy
measureVelocity
assessAlignment
kpi_alpha
kpi_beta
kpi_gamma
synergy_table
Productivity: 100%%. Finally, someone who gets it.
```

This immediately gives us some useful clues:

* Three validation functions
* Three KPI-related data arrays
* A substitution table
* A success condition

So instead of brute-forcing the employee ID, we can reverse each validation function independently.

---

# 2. Understanding `main`

Using `objdump`:

```bash
objdump -d -M intel kpiman
```

Looking at `main`, we can determine the following logic:

1. Reads user input using `fgets()`.
2. Removes the trailing newline using `strcspn()`.
3. Requires the input length to be exactly `0x2c` bytes.

```text
0x2c = 44
```

4. Calls three validation functions:

```c
calculateSynergy(input);
measureVelocity(input);
assessAlignment(input);
```

5. Each function returns either `0` or `1`.
6. Their results are added together.
7. Only when the result is `3` does the program print the 100% productivity message.

Conceptually:

```c
score =
    calculateSynergy(input) +
    measureVelocity(input) +
    assessAlignment(input);

if (score == 3) {
    printf("Productivity: 100%%. Finally, someone who gets it.");
    printf("Promotion code: %s", input);
}
```

Therefore, we need an input that satisfies **all three checks**.

The important observation is that the checks operate on separate portions of the 44-byte input.

```text
Bytes 0  - 14  → calculateSynergy
Bytes 15 - 29  → measureVelocity
Bytes 30 - 43  → assessAlignment
```

So the problem can be solved in three independent stages.

---

# 3. `calculateSynergy` — Bytes 0–14

The first function checks the first 15 bytes.

A simplified version of the logic is:

```c
for (i = 0; i <= 14; i++) {
    key = (i * 8 - i) + 42;

    if ((input[i] ^ (uint8_t)key) != kpi_alpha[i])
        return 0;
}

return 1;
```

Since:

```text
i * 8 - i = i * 7
```

the key is:

```text
key = i * 7 + 42
```

The comparison is:

```text
input[i] XOR key == kpi_alpha[i]
```

XOR is reversible, so:

```text
input[i] = kpi_alpha[i] XOR key
```

Therefore:

```python
for i in range(15):
    input_bytes[i] = kpi_alpha[i] ^ ((i * 7 + 42) & 0xff)
```

The `kpi_alpha` array can be recovered from `.rodata`.

---

# 4. `measureVelocity` — Bytes 15–29

The second function handles bytes `15` through `29`.

Its logic is:

```c
for (i = 15; i <= 29; i++) {
    sum = input[i] + input[i - 1];

    if (sum != kpi_beta[i - 15])
        return 0;
}

return 1;
```

Here, every byte depends on the previous byte.

Rearranging:

```text
input[i] + input[i-1] = kpi_beta[i-15]
```

Therefore:

```text
input[i] = kpi_beta[i-15] - input[i-1]
```

We already know `input[14]` from the first stage, so we can solve the remaining bytes sequentially.

```python
for i in range(15, 30):
    input_bytes[i] = (
        kpi_beta[i - 15] - input_bytes[i - 1]
    ) & 0xff
```

This gives us bytes `15–29`.

---

# 5. `assessAlignment` — Bytes 30–43

The final function checks bytes `30` through `43`.

The relevant logic is:

```c
for (i = 30; i <= 43; i++) {
    if (synergy_table[input[i]] != kpi_gamma[i - 30])
        return 0;
}

return 1;
```

Here, `synergy_table` is a 256-byte substitution table.

The relationship is:

```text
synergy_table[input[i]] = kpi_gamma[i-30]
```

The table contains a complete permutation of values `0x00–0xff`, meaning every output has exactly one corresponding input.

Therefore, we can construct an inverse lookup table:

```python
inv_sbox = {
    value: index
    for index, value in enumerate(synergy_table)
}
```

Then:

```python
for i in range(30, 44):
    input_bytes[i] = inv_sbox[kpi_gamma[i - 30]]
```

This recovers the final 14 bytes.

---

# 6. Solving Everything

The four relevant `.rodata` regions can be dumped with:

```bash
objdump -s -j .rodata kpiman
```

After extracting:

* `kpi_alpha`
* `kpi_beta`
* `kpi_gamma`
* `synergy_table`

we can reproduce the validation logic in Python.

```python
# kpi_alpha stage — bytes 0-14
for i in range(15):
    input_bytes[i] = kpi_alpha[i] ^ ((i * 7 + 42) & 0xff)


# kpi_beta stage — bytes 15-29
# Starts from the previously recovered input[14]
for i in range(15, 30):
    input_bytes[i] = (
        kpi_beta[i - 15] - input_bytes[i - 1]
    ) & 0xff


# synergy_table stage — bytes 30-43
# Build inverse substitution table
inv_sbox = {
    value: index
    for index, value in enumerate(synergy_table)
}

for i in range(30, 44):
    input_bytes[i] = inv_sbox[kpi_gamma[i - 30]]
```

Finally:

```python
flag = bytes(input_bytes)
print(flag.decode())
```

Output:

```text
brunner{y0ur_kp1s_ar3_n0t_l00king_gr8_buddy}
```

---

# 7. Verification

Let's test the recovered value against the binary:

```bash
echo "brunner{y0ur_kp1s_ar3_n0t_l00king_gr8_buddy}" | ./kpiman
```

Output:

```text
BrunnerCorp KPIman v3.1
Enter employee ID: Analyzing synergy...
Productivity: 100%. Finally, someone who gets it.
Promotion code: brunner{y0ur_kp1s_ar3_n0t_l00king_gr8_buddy}
```

The recovered input successfully passes all three checks.

```text
Productivity: 100%
```

---

# Flag

```text
brunner{y0ur_kp1s_ar3_n0t_l00king_gr8_buddy}
```

---

# Takeaways

This challenge initially looks more complicated because the binary has three separate KPI checks, but each one is based on a simple reversible transformation.

### `calculateSynergy`

A simple XOR transformation:

```text
cipher = plaintext XOR key
```

which can be reversed using XOR again.

### `measureVelocity`

A chained addition:

```text
input[i] + input[i-1] = target
```

which can be solved sequentially once the first byte is known.

### `assessAlignment`

A substitution box:

```text
SBOX[input] = target
```

which can be reversed by constructing the inverse S-box.

The biggest shortcut was noticing that the binary was **not stripped**. The surviving symbol names immediately revealed the intended structure:

```text
calculateSynergy
measureVelocity
assessAlignment
kpi_alpha
kpi_beta
kpi_gamma
synergy_table
```

Rather than brute-forcing a 44-byte input, we can split the problem into three small, independently invertible constraints.

**No real cryptography — just reverse engineering the transformations.** 🔍
