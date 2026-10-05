# Go Go Decompile — Reversing / Easy

**Category:** Reversing
**Points:** 100
**Author:** Quack

## Challenge

> Uh oh, I've been crunching numbers all month in `Go Go BudgetMaster`, but the cleaning crew accidentally threw out my license key Post-it. Now management's breathing down my neck about the budget.
> Maybe I can use that magic dragon program our security guy keeps blabbing about at lunch?

We're given a single binary, `go_go_budgetmaster`, and need to recover a license key to make it print a success message.

## Recon

```console
$ file go_go_budgetmaster
go_go_budgetmaster: ELF 64-bit LSB executable, x86-64, version 1 (SYSV),
statically linked, with debug_info, not stripped
```

Statically-linked, unstripped Go binary. "The magic dragon program" is Ghidra — but a Go-aware disassembler isn't strictly necessary here; `objdump` gets us there just fine.

A quick symbol dump confirms this is a plain Go program with no custom packages — only `main.main` survived as a distinct symbol, since the Go compiler/linker inlined everything else straight into it:

```console
$ nm go_go_budgetmaster | grep ' T main\.'
00000000004a1f80 T main.main
```

So the entire challenge logic lives in one function.

## Static Analysis

Disassembling `main.main`:

```console
$ objdump -d --disassemble=main.main -M intel go_go_budgetmaster
```

Reading through it, the control flow is refreshingly linear — no hidden branches, no anti-debug tricks:

1. **Print a prompt** via `os.Stdout.WriteString` using a string at `0x4c6fe6` (15 bytes).
2. **Read one line** from stdin with `bufio.NewScanner` / `Scan` / `Text`.
3. **Base64-decode** a hardcoded blob at `0x4cc9e8` (40 bytes) using `encoding/base64.StdEncoding.Decode`.
4. **Compare** the decoded bytes against your input via `runtime.memequal`.
5. Branch to one of two hardcoded strings depending on the result (`0x4cca10` on success, `0x4cc72a` on failure).

No hashing, no obfuscation, no key-derivation — just `input == base64_decode(blob)`.

## Pulling the Strings

Since the binary isn't stripped and nothing is encrypted, the blob can be read straight out of `.rodata` by mapping virtual addresses to file offsets (`ELF_vaddr - load_bias`, confirmed against `readelf -S`):

```python
with open('go_go_budgetmaster', 'rb') as f:
    data = f.read()

def read_str(vaddr, length):
    off = vaddr - 0x400000
    return data[off:off+length]

print(read_str(0x4c6fe6, 0xf))   # prompt
print(read_str(0x4cc9e8, 0x28))  # base64 blob
print(read_str(0x4cca10, 0x28))  # success message
print(read_str(0x4cc72a, 0x27))  # failure message
```

Output:

```
prompt:  b'Go Go License? '
blob:    b'YnJ1bm5lcntnMF9kM2MwbXAxbDNkX2cwX2Jycn0='
success: b'Correct!\nThis is way better than Excel!\n'
failure: b'Incorrect!\nAre you sure you work here?\n'
```

## Decoding the Flag

```console
$ echo YnJ1bm5lcntnMF9kM2MwbXAxbDNkX2cwX2Jycn0= | base64 -d
brunner{g0_d3c0mp1l3d_g0_brr}
```

## Verification

```console
$ echo 'brunner{g0_d3c0mp1l3d_g0_brr}' | ./go_go_budgetmaster
Go Go License? Correct!
This is way better than Excel!
```

## Flag

```
brunner{g0_d3c0mp1l3d_g0_brr}
```

## Takeaways

- Unstripped Go binaries dump most of their logic into `main.main`, which makes a quick `objdump -d --disassemble=main.main` a great first move for easy reversing challenges.
- Always check for a plain equality comparison (`runtime.memequal`) before assuming you need to brute-force or reverse a hash — static strings in `.rodata` can often just be read out directly.
- `strings` + section-offset math beats firing up a full disassembler when the binary isn't stripped and the logic is this linear.
