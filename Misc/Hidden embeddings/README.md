# Hidden Embeddings — Writeup

**Category:** ML
**Difficulty:** Medium
**Points:** 100

---

## Challenge

> Our competitors have been baking a new AI model, and I overheard that they've hidden an important 35-character trade secret inside. Unfortunately, they've obfuscated the model by shuffling the few real layers and introducing a bunch of fake layers with random bias to keep it secure.

**Files provided:**

* `model.py` — Defines `HiddenEmbeddingNet`, a 16-layer MLP
* `run.py` — Loads the checkpoint and runs all 16 layers in order
* `model.safetensors` — Contains the model weights

The interesting part of `model.py` is:

```python
NUM_LAYERS = 16

class HiddenEmbeddingNet(nn.Module):
    def forward(self, x):
        # Gotta use all layers and in order... right?
        for layer in self.layers:
            x = F.relu(layer(x))
        return x
```

The comment is the main hint:

> **"Gotta use all layers and in order... right?"**

This suggests that using every layer in the given order may be the intended misdirection.

---

## Solution

### 1. Inspect the Weight Shapes

Since the model uses a `safetensors` checkpoint, the first step is to inspect the dimensions of every layer.

```python
from safetensors.torch import load_file

sd = load_file("model.safetensors")

for k, v in sd.items():
    print(k, tuple(v.shape))
```

The important dimensions are:

```text
layers.0:   35 → 47
layers.1:   47 → 54
layers.2:   54 → 51
layers.3:   51 → 44
layers.4:   44 → 55
layers.5:   55 → 31
layers.6:   31 → 51
layers.7:   51 → 18
layers.8:   18 → 35
layers.9:   35 → 35
layers.10:  35 → 35
layers.11:  35 → 35
layers.12:  35 → 35
layers.13:  35 → 59
layers.14:  59 → 56
layers.15:  56 → 13
```

---

## 2. Identify the Trap

At first glance, the layers look completely valid.

In fact, the dimensions line up perfectly:

```text
35 → 47 → 54 → 51 → 44 → 55 → 31 → 51
   → 18 → 35 → 35 → 35 → 35 → 59 → 56 → 13
```

So `run.py` can execute the entire 16-layer network without producing a shape mismatch.

But there is a problem.

The network starts with **35 dimensions** and ends with **13 dimensions**.

The challenge tells us that the hidden secret is **35 characters long**.

That strongly suggests the real embedding should map:

```text
35 → 35
```

rather than:

```text
35 → 13
```

Therefore, the long 16-layer chain is likely a decoy.

---

## 3. Find the Real Layers

We can filter the checkpoint for layers whose weight matrices are exactly `35 × 35`.

```python
square = [
    k.split(".")[1]
    for k, v in sd.items()
    if k.endswith(".weight") and v.shape == (35, 35)
]

print(square)
```

Output:

```text
['9', '10', '11', '12']
```

So only four layers have the required dimensions:

```text
layers.9
layers.10
layers.11
layers.12
```

These are the most likely candidates for the actual hidden embedding network.

---

## 4. Brute-Force the Layer Order

There are only four candidate layers.

That means the number of possible permutations is:

```text
4! = 24
```

So instead of trying to reverse the entire neural network, we can simply test every possible ordering.

Use a one-hot input vector:

```python
x0 = torch.zeros(35)
x0[0] = 1.0
```

Then apply each permutation:

```python
import itertools
import torch
import torch.nn.functional as F

real = ["9", "10", "11", "12"]

x0 = torch.zeros(35)
x0[0] = 1.0


def apply(order, x):
    for idx in order:
        x = F.relu(
            F.linear(
                x,
                sd[f"layers.{idx}.weight"],
                sd[f"layers.{idx}.bias"]
            )
        )
    return x


for perm in itertools.permutations(real):
    out = apply(perm, x0).tolist()

    if all(32 <= round(v) < 127 for v in out):
        decoded = "".join(chr(round(v)) for v in out)
        print(perm, "->", decoded)
```

There are only 24 possibilities, so this finishes almost instantly.

---

## 5. Recover the Secret

The correct permutation is:

```text
('10', '9', '12', '11')
```

which produces:

```text
brunner{0hh_n0_y0u_f0und_my_s3cr3t}
```

### Flag

```text
brunner{0hh_n0_y0u_f0und_my_s3cr3t}
```

---

## Full Solver

```python
import itertools
import torch
import torch.nn.functional as F
from safetensors.torch import load_file


sd = load_file("model.safetensors")

# Find 35x35 layers
real = [
    k.split(".")[1]
    for k, v in sd.items()
    if k.endswith(".weight") and v.shape == (35, 35)
]

print("[+] Candidate layers:", real)


# One-hot input
x0 = torch.zeros(35)
x0[0] = 1.0


def apply(order, x):
    for idx in order:
        x = F.relu(
            F.linear(
                x,
                sd[f"layers.{idx}.weight"],
                sd[f"layers.{idx}.bias"]
            )
        )
    return x


# Try every possible ordering
for perm in itertools.permutations(real):
    out = apply(perm, x0).tolist()

    # Printable ASCII
    if all(32 <= round(v) < 127 for v in out):
        decoded = "".join(chr(round(v)) for v in out)

        print(f"[+] {perm} -> {decoded}")
```

Output:

```text
[+] ('10', '9', '12', '11') -> brunner{0hh_n0_y0u_f0und_my_s3cr3t}
```

---

## Why It Works

The important observation is that the model's **dimensional structure leaks the intended architecture**.

The full network appears to be valid because every consecutive layer happens to have compatible dimensions:

```text
35 → 47 → 54 → ... → 56 → 13
```

But the challenge explicitly tells us the secret is **35 characters**.

The only layers that preserve that dimensionality are:

```text
35 → 35
35 → 35
35 → 35
35 → 35
```

That reduces the problem from analyzing a 16-layer neural network to testing just:

```text
4! = 24
```

possible layer orders.

The correct order then directly produces printable ASCII characters representing the flag.

---

## Takeaway

> **Dimensional compatibility doesn't imply correctness.**

The 16-layer chain being executable is part of the misdirection.

The key clues were:

1. The secret is **35 characters** long.
2. The model input is **35-dimensional**.
3. Only four layers are `35 × 35`.
4. Four layers have only **24 possible permutations**.
5. One permutation produces valid printable ASCII.

So rather than trusting the model's `forward()` implementation, we inspect the underlying weights and reconstruct the meaningful sub-network.

**Flag:** `brunner{0hh_n0_y0u_f0und_my_s3cr3t}`
