# Go Go Decompile

**Category:** Reversing
**Difficulty:** Easy
**Points:** 100
**Author:** Quack

## Challenge

> Uh oh, I've been crunching numbers all month in `Go Go BudgetMaster`, but the cleaning crew accidentally threw out my license key Post-it. Now management's breathing down my neck about the budget. Maybe I can use that magic dragon program our security guy keeps blabbing about at lunch?

We are given a single binary:

```text
go_go_budgetmaster
```

The goal is to recover the license key that makes the program print the success message.

---

## Recon

First, identify the binary:

```bash
file go_go_budgetmaster
```

Output:

```text
go_go_budgetmaster: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
statically linked, with debug_info, not stripped
```

This is a **statically linked, unstripped Go binary**.

The challenge mentions a "magic dragon program", which is likely a reference to **Ghidra**. However, because the binary is unstripped and contains debug information, basic tools such as `nm` and `objdump` are enough.

Let's inspect the symbols:

```bash
nm go_go_budgetmaster | grep ' T main\.'
```

Output:

```text
00000000004a1f80 T main.main
```

The main challenge logic is contained inside `main.main`.

---

## Static Analysis

Disassemble `main.main`:

```bash
objdump -d --disassemble=main.main -M intel go_go_budgetmaster
```

The control flow is fairly straightforward.

The important operations are:

1. Print the license prompt.
2. Read one line from standard input.
3. Base64-decode a hardcoded string.
4. Compare the decoded bytes with the supplied input.
5. Print either the success or failure message.

The comparison is performed using:

```text
runtime.memequal
```

So there is no password hashing, encryption, or complicated key derivation.

Conceptually, the program is doing:

```text
input == base64_decode(hardcoded_blob)
```

Therefore, recovering the license key only requires extracting the Base64 string and decoding it.

---

## Extracting the Strings

The hardcoded strings are stored in `.rodata`.

The relevant virtual addresses are:

```text
Prompt:   0x4c6fe6
Blob:     0x4cc9e8
Success:  0x4cca10
Failure:  0x4cc72a
```

Since the binary uses a load bias of `0x400000`, the file offset can be calculated as:

```text
file_offset = virtual_address - 0x400000
```

A small Python script can extract the strings:

```python
with open("go_go_budgetmaster", "rb") as f:
    data = f.read()


def read_str(vaddr, length):
    offset = vaddr - 0x400000
    return data[offset:offset + length]


print(read_str(0x4c6fe6, 0xf))
print(read_str(0x4cc9e8, 0x28))
print(read_str(0x4cca10, 0x28))
print(read_str(0x4cc72a, 0x27))
```

Output:

```text
prompt:  b'Go Go License? '
blob:    b'YnJ1bm5lcntnMF9kM2MwbXAxbDNkX2cwX2Jycn0='
success: b'Correct!\nThis is way better than Excel!\n'
failure: b'Incorrect!\nAre you sure you work here?\n'
```

The important value is:

```text
YnJ1bm5lcntnMF9kM2MwbXAxbDNkX2cwX2Jycn0=
```

---

## Decoding the Base64

Decode it using:

```bash
echo 'YnJ1bm5lcntnMF9kM2MwbXAxbDNkX2cwX2Jycn0=' | base64 -d
```

Result:

```text
brunner{g0_d3c0mp1l3d_g0_brr}
```

---

## Verification

Run the binary with the recovered license key:

```bash
echo 'brunner{g0_d3c0mp1l3d_g0_brr}' | ./go_go_budgetmaster
```

Output:

```text
Go Go License? Correct!
This is way better than Excel!
```

The key is correct.

---

## Flag

```text
brunner{g0_d3c0mp1l3d_g0_brr}
```

---

## Takeaways

* An **unstripped Go binary** can expose useful symbols and debugging information.
* `nm` is useful for quickly identifying functions such as `main.main`.
* `objdump` can be enough for simple reversing challenges without opening a full GUI disassembler.
* Always look for direct comparisons such as `runtime.memequal` before assuming a password is hashed or encrypted.
* Hardcoded data in `.rodata` can often be extracted directly from the binary.
* Base64 is **encoding, not encryption**. If a Base64 blob is hardcoded, decoding it may immediately reveal the secret.
* For simple binaries, a combination of `file`, `nm`, `objdump`, `strings`, and a small Python script can be faster than using a full reverse-engineering framework.

## Tools Used

* `file`
* `nm`
* `objdump`
* Python 3
* `base64`
* Ghidra (optional)
