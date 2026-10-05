# KPWhy — Writeup

**Category:** Reverse Engineering
**Difficulty:** Easy-Medium
**Flag:** `brunner{y0ur_kp1s_ar3_n0t_l00king_gr8_buddy}`

## Challenge

We're given a binary, `kpiman`, that prompts for an "employee ID" and reports a
productivity score. Exactly one input yields 100%.

```
$ ./kpiman
BrunnerCorp KPIman v3.1
Enter employee ID: AAAA
Analyzing synergy...
Productivity: 0%. Have you considered a career in cake tasting?
```

## Triage

```
$ file kpiman
kpiman: ELF 64-bit LSB executable, x86-64, ... not stripped
```

Not stripped, so symbol names survive. `strings` immediately shows the
interesting function names and format strings:

```
calculateSynergy
measureVelocity
assessAlignment
kpi_alpha
kpi_beta
kpi_gamma
synergy_table
Productivity: 100%%. Finally, someone who gets it.
```

That's a strong hint: three check functions, three "scoring" globals, and a
lookup table.

## Reading `main`

Disassembling with `objdump -d -M intel kpiman` and looking at `main`:

1. Reads a line with `fgets` into a 256-byte stack buffer, strips the
   trailing newline (`strcspn`).
2. Requires `strlen(input) == 0x2c` (44) — otherwise prints
   "Wrong badge length." and exits.
3. Calls `calculateSynergy(input)`, `measureVelocity(input)`,
   `assessAlignment(input)` — each returns 0 or 1.
4. Sums the three results. Only if the sum is **3** (i.e. all three functions
   return 1) does it print `Productivity: 100%%` and the promotion code
   (simply `printf("%s", input)` — the flag *is* the valid input).

So the job is: find the unique 44-byte string that satisfies all three
checks independently. Each function only touches a disjoint 14–15 byte slice
of the input, so the three checks can be solved separately and concatenated.

## `calculateSynergy` — bytes 0–14 (15 bytes)

```c
for (i = 0; i <= 14; i++) {
    key = (i*8 - i) + 42;          // == i*7 + 42, masked to a byte
    if ((input[i] ^ (uint8_t)key) != kpi_alpha[i])
        return 0;
}
return 1;
```

`kpi_alpha` is a 15-byte array taken straight from `.rodata` at `0x402020`.
This is a simple keyed XOR, trivially invertible:

```
input[i] = kpi_alpha[i] ^ ((i*7 + 42) & 0xFF)
```

## `measureVelocity` — bytes 15–29 (15 bytes)

```c
for (i = 15; i <= 29; i++) {
    sum = input[i] + input[i-1];
    if (sum != kpi_beta[i-15])      // kpi_beta is an int[15] array
        return 0;
}
return 1;
```

`kpi_beta` (at `0x402040`) gives the *sum* of each byte with its predecessor.
Since `input[14]` is already known from the previous stage, this chains
forward byte by byte:

```
input[i] = kpi_beta[i-15] - input[i-1]   for i = 15..29
```

## `assessAlignment` — bytes 30–43 (14 bytes)

```c
for (i = 30; i <= 43; i++) {
    if (synergy_table[input[i]] != kpi_gamma[i-30])
        return 0;
}
return 1;
```

`synergy_table` (at `0x402080`... actually `0x4020a0`) is a 256-byte
substitution box used as `table[input_byte] -> output_byte`. Since it's a
bijection (full 0–255 permutation), it can be inverted into a lookup
`output_byte -> input_byte`, and each target byte in `kpi_gamma` (14 bytes,
at `0x402080`) maps straight back to the original input byte.

## Solving

Dumped the four `.rodata` blobs with `objdump -s -j .rodata` and reimplemented
the three inverses in Python:

```python
# kpi_alpha stage (bytes 0-14)
for i in range(15):
    input_bytes[i] = kpi_alpha[i] ^ ((i*7 + 42) & 0xFF)

# kpi_beta stage (bytes 15-29), chained off input[14]
for i in range(15, 30):
    input_bytes[i] = (kpi_beta[i-15] - input_bytes[i-1]) & 0xFF

# synergy_table stage (bytes 30-43), inverse S-box lookup
inv_sbox = {v: k for k, v in enumerate(synergy_table)}
for i in range(30, 44):
    input_bytes[i] = inv_sbox[kpi_gamma[i-30]]
```

Concatenating the 44 bytes decodes cleanly as ASCII:

```
brunner{y0ur_kp1s_ar3_n0t_l00king_gr8_buddy}
```

## Verification

```
$ echo "brunner{y0ur_kp1s_ar3_n0t_l00king_gr8_buddy}" | ./kpiman
BrunnerCorp KPIman v3.1
Enter employee ID: Analyzing synergy...
Productivity: 100%. Finally, someone who gets it.
Promotion code: brunner{y0ur_kp1s_ar3_n0t_l00king_gr8_buddy}
```

100%, as promised.

## Takeaways

- The three "KPI" checks look intimidating together but are each a
  textbook-simple, *independently invertible* transform (XOR with a
  keystream, a running-sum chain, and a substitution box) operating on
  disjoint slices of the input — classic easy-medium RE design: lots of
  surface area, no real cryptographic difficulty once decomposed.
- Keeping the binary unstripped made the intended structure obvious from
  symbol names alone, which is a good signal to immediately split the
  problem into independent sub-constraints rather than brute-forcing.
