# Activating Neurons — Writeup

**Category:** Misc
**Difficulty:** Easy
**Author:** MiKi

## Challenge

> My colleague in IT made a Neural Network to remember the most important ingredients in Brunsviger, but he made the network completely passive and linear. Can you *rectify* the situation and recover the secret?

We are given a Python script containing a small PyTorch neural network called `BrunsvigerNet`.

The script creates a dummy input and converts the network's output values into ASCII characters:

```python
dummy_input = torch.zeros(4)

output = model(dummy_input)
flag = "".join([chr(int(round(val.item()))) for val in output])
```

The goal is therefore to make the network produce values corresponding to ASCII characters.

---

## Initial Analysis

The network contains two linear layers:

```python
self.input_layer = nn.Linear(4, 4)
self.hidden_layer = nn.Linear(4, 70)
```

The input layer has a predefined bias:

```python
self.input_layer.bias = nn.Parameter(
    torch.tensor([-0.797, -0.047, 0.527, 1.965])
)
```

The hidden layer contains 70 rows of weights and 70 bias values.

The important part is the `forward()` function:

```python
def forward(self, x):
    x = self.input_layer(x)

    # Something seems to be missing?

    return x
```

The comment is a strong hint that part of the network's forward pass has been removed.

---

## Understanding the Input

The challenge uses:

```python
dummy_input = torch.zeros(4)
```

Because the input is a zero vector, the first linear layer produces essentially its bias:

$$
W x + b = b
$$

Therefore:

```text
[-0.797, -0.047, 0.527, 1.965]
```

is passed to the next layer.

The second layer has dimensions:

```text
4 → 70
```

which is interesting because 70 output values are enough to encode a 70-character ASCII string.

---

## The Important Clue

The challenge description says the neural network is:

> "completely passive and linear"

This is the key.

Initially, the wording **"rectify"** can make you think of a ReLU activation:

```python
x = F.relu(x)
```

However, that turns out to be a red herring.

If ReLU is added:

```python
def forward(self, x):
    x = self.input_layer(x)
    x = F.relu(x)
    x = self.hidden_layer(x)
    return x
```

the output becomes:

```text
^swklirznkcd2o`a3]ixqcsi/bl.9r`/mr,rq7puc/liv3a4/kt]/9ah-t2`2mb`9uf3t
```

This is printable, but clearly not the flag.

The problem is that ReLU destroys the negative values from the first layer:

```text
[-0.797, -0.047, 0.527, 1.965]
```

becomes:

```text
[0, 0, 0.527, 1.965]
```

Since the challenge explicitly describes the network as **linear**, those negative values must be preserved.

---

## Restoring the Missing Layer

The intended forward pass is simply both linear layers:

```python
def forward(self, x):
    x = self.input_layer(x)
    x = self.hidden_layer(x)

    return x
```

So the complete architecture is:

```text
              Linear 4 → 4
                    │
                    ▼
              Linear 4 → 70
                    │
                    ▼
               ASCII values
                    │
                    ▼
                  Flag
```

No activation function is required.

---

## Running the Solution

After modifying the `forward()` function, run:

```bash
python3 activating_neurons.py
```

The 70 output values correspond to ASCII characters.

The beginning of the output is:

```text
98 114 117 110 110 101 114 123 ...
```

Converting these decimal values to ASCII gives:

```text
brunner{...
```

Continuing the conversion reveals the complete flag.

---

## Final Flag

```text
brunner{ml_c4n_b3_fun_th3_m05t_1mp0rt4nt_1ngr3d13nt_15_l0v3_4nd_5ug4r}
```

## Key Takeaway

The challenge is based on recognizing that the provided neural network is incomplete.

The important observation is that the network is explicitly described as **linear**. Therefore, adding ReLU is actually incorrect because it changes the first layer's negative values.

The missing operation was simply the second linear layer:

```python
x = self.hidden_layer(x)
```

Once the two linear transformations are chained together, the resulting 70 values directly represent the flag in ASCII.

**Flag:**

```text
brunner{ml_c4n_b3_fun_th3_m05t_1mp0rt4nt_1ngr3d13nt_15_l0v3_4nd_5ug4r}
```
