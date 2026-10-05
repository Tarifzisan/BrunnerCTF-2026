# Roadmap — Writeup

**Category:** Reverse Engineering
**Difficulty:** Medium
**Files:** `Dockerfile`, `compose.yaml`, `default.conf`, `roadmap-badges.conf`

## Flag

```text
brunner{c0rp0r4t3_r04dm4p_t0_ng1nx_h34rt}
```

---

## TL;DR

The challenge is an Nginx configuration puzzle.

The server checks the requested URL path against a hidden **41-character secret** using a chain of `map` directives. The configuration contains:

* Character extraction maps
* A character-to-hex substitution layer
* A linked chain of checkpoint states
* A final `CLEARED` condition

There is no need to brute-force the server. Everything required to recover the flag is statically encoded in the Nginx configuration.

The recovered path is:

```text
/brunner{c0rp0r4t3_r04dm4p_t0_ng1nx_h34rt}
```

Requesting it causes Nginx to return the **Access cleared** page.

---

# Challenge Setup

The provided files are:

```text
Dockerfile
compose.yaml
default.conf
roadmap-badges.conf
```

`Dockerfile` and `compose.yaml` simply build and run the Nginx server.

The interesting logic is inside:

```text
default.conf
roadmap-badges.conf
```

---

# Step 1 — Extracting Characters

The configuration first extracts the requested URL path.

```nginx
map $uri $route {
    default "";
    "~^/(?<s>.*)$" $s;
}

map $route $route_len_ok {
    default 0;
    "~^.{41}$" 1;
}
```

`$route` contains the requested path without the leading `/`.

For example:

```text
/brunner{...}
```

becomes:

```text
brunner{...}
```

The second map ensures that the path is exactly **41 characters** long.

---

## Character Extraction Maps

There are 41 different `wp_*` variables.

For example:

```nginx
map $route $wp_8f96 {
    default "";
    "~^.{8}(?<c>.)" $c;
}
```

This extracts the character at offset `8`.

Conceptually:

```text
$route
  |
  +--> offset 0 --> wp_xxxx
  +--> offset 1 --> wp_xxxx
  +--> offset 2 --> wp_xxxx
  ...
  +--> offset 40 --> wp_xxxx
```

The variable names are randomized, but each one corresponds to a specific position in the 41-character route.

Therefore, the first thing to recover is:

```text
wp_XXXX -> character offset
```

---

# Step 2 — The Badge Substitution Layer

Each extracted character is passed through the same substitution table.

For example:

```nginx
map $wp_8100 $badge_2bae {
    include roadmap-badges.conf;
}
```

The included file contains mappings such as:

```text
"a" "7f";
"b" "59";
...
"_" "f5";
"{" "34";
"}" "3d";
```

So the system effectively performs:

```text
character -> 2-digit hexadecimal badge
```

For example:

```text
a -> 7f
b -> 59
_ -> f5
{ -> 34
} -> 3d
```

Every character therefore has a unique encoded representation.

The randomized variable names make the configuration look much more complicated than it actually is.

---

# Step 3 — Understanding the Checkpoint Chain

The interesting part is the chain of `cp_*` variables.

A typical checkpoint looks like:

```nginx
map $badge_a8e4 $cp_8a32 {
    default "DETOUR";
    "0a" "chk_f6ca";
}
```

The first checkpoint checks a single badge.

If the badge is correct:

```text
badge_a8e4 = 0a
```

then:

```text
cp_8a32 = chk_f6ca
```

Otherwise:

```text
cp_8a32 = DETOUR
```

---

## Subsequent Checkpoints

The next checkpoints depend on both:

1. The previous checkpoint state
2. The current badge

For example:

```nginx
map "${cp_8a32}:${badge_fcf8}" $cp_a491 {
    default "DETOUR";
    "chk_f6ca:06" "chk_fc9f";
}
```

This means:

```text
previous checkpoint = chk_f6ca
AND
current badge = 06
```

must both be correct.

If either value is wrong:

```text
DETOUR
```

is produced.

The `DETOUR` state cannot satisfy the following checkpoints, so a single incorrect character causes the entire chain to fail.

---

# Step 4 — Following the Chain Backwards

The final checkpoint looks like:

```nginx
map "${cp_1be7}:${badge_6680}" $cp_199f {
    default "DETOUR";
    "chk_3a12:0a" "CLEARED";
}
```

Therefore, to reach:

```text
CLEARED
```

we know the previous state must be:

```text
chk_3a12
```

and the required badge must be:

```text
0a
```

Instead of trying every possible 41-character string, we can work backwards.

The chain looks conceptually like:

```text
ENTRY
  |
  v
chk_xxxx
  |
  v
chk_xxxx
  |
  v
chk_xxxx
  |
  ...
  |
  v
CLEARED
```

By reversing the graph, we can determine:

```text
checkpoint
    ↓
required badge
    ↓
character position
    ↓
actual character
```

---

# Step 5 — Recovering the Flag Automatically

The following Python script parses the Nginx configuration and reconstructs the route.

```python
import re

conf = open("default.conf").read()
badges_conf = open("roadmap-badges.conf").read()

# Parse character -> hexadecimal badge mappings
badge_map = dict(
    re.findall(
        r'"(.)"\s+"([0-9a-f]{2})"',
        badges_conf
    )
)

# Reverse the mapping:
# hexadecimal badge -> character
inv_badge = {
    value: key
    for key, value in badge_map.items()
}


# --------------------------------------------------
# Parse wp_* -> character offset
# --------------------------------------------------

wp_offset = {
    m.group(1): int(m.group(2))
    for m in re.finditer(
        r'map \$route \$wp_(\w+) '
        r'\{ default ""; "~\^\.\{(\d+)\}',
        conf
    )
}


# --------------------------------------------------
# Parse badge_* -> wp_*
# --------------------------------------------------

badge_to_wp = {
    m.group(2): m.group(1)
    for m in re.finditer(
        r'map \$wp_(\w+) \$badge_(\w+) '
        r'\{ include roadmap-badges\.conf; \}',
        conf
    )
}


# --------------------------------------------------
# Parse checkpoint maps
# --------------------------------------------------

cp_edges = {}


# Checkpoints that depend on a previous checkpoint
for m in re.finditer(
    r'map "\$\{cp_(\w+)\}:\$\{badge_(\w+)\}" '
    r'\$cp_(\w+) \{ default "DETOUR"; '
    r'"(chk_\w+):([0-9a-f]{2})" "(\w+)"; \}',
    conf
):
    prev_cp, badge, cp, req_chk, req_hex, result_chk = m.groups()

    cp_edges[cp] = {
        "prev_cp": prev_cp,
        "badge": badge,
        "req_hex": req_hex,
        "result_chk": result_chk,
    }


# Entry-point checkpoint
for m in re.finditer(
    r'map \$badge_(\w+) \$cp_(\w+) '
    r'\{ default "DETOUR"; '
    r'"([0-9a-f]{2})" "(\w+)"; \}',
    conf
):
    badge, cp, req_hex, result_chk = m.groups()

    cp_edges[cp] = {
        "prev_cp": None,
        "badge": badge,
        "req_hex": req_hex,
        "result_chk": result_chk,
    }


# --------------------------------------------------
# Find the checkpoint that produces CLEARED
# --------------------------------------------------

final = next(
    cp
    for cp, edge in cp_edges.items()
    if edge["result_chk"] == "CLEARED"
)


# --------------------------------------------------
# Walk backwards through the checkpoint chain
# --------------------------------------------------

route = [None] * 41

cur = final

while cur is not None:
    edge = cp_edges[cur]

    # Determine which route offset this badge represents
    offset = wp_offset[
        badge_to_wp[edge["badge"]]
    ]

    # Convert required hexadecimal badge back to character
    route[offset] = inv_badge[edge["req_hex"]]

    # Continue backwards
    cur = edge["prev_cp"]


flag = "".join(route)

print(flag)
```

The script outputs:

```text
brunner{c0rp0r4t3_r04dm4p_t0_ng1nx_h34rt}
```

---

# Step 6 — Verification

The recovered value is used directly as the URL path:

```text
/brunner{c0rp0r4t3_r04dm4p_t0_ng1nx_h34rt}
```

The server can then be tested with:

```bash
curl 'http://localhost:3000/brunner{c0rp0r4t3_r04dm4p_t0_ng1nx_h34rt}'
```

Expected response:

```html
<!doctype html>
<meta charset=utf-8>
<title>Cleared</title>

<h1>✅ Access cleared, stakeholder.</h1>
<p>You walked the corporate roadmap to the letter.
Welcome aboard Brunnerne Inc.™</p>
```

This confirms that:

```text
$cp_199f == "CLEARED"
```

and:

```text
$route_len_ok == 1
```

Therefore:

```text
$access == 1
```

---

# Flag

```text
brunner{c0rp0r4t3_r04dm4p_t0_ng1nx_h34rt}
```

---

# Takeaways

### 1. The challenge is a config-based maze

The challenge doesn't rely on application code or a traditional web vulnerability.

Instead, Nginx's `map` directive is used to construct a state machine.

```text
URL character
     ↓
wp_* extraction
     ↓
badge_* substitution
     ↓
checkpoint state
     ↓
next checkpoint
     ↓
...
     ↓
CLEARED
```

### 2. No brute force is required

Although the flag is 41 characters long, there is no need to brute-force the URL.

All required information is already present in the configuration.

The important step is recognizing that the `cp_*` maps form a graph.

### 3. The randomized variable names are the main obfuscation

Names such as:

```text
wp_8f96
badge_2bae
cp_8a32
```

look random, but they are simply identifiers connecting different parts of the configuration.

Once those relationships are parsed, the puzzle becomes a straightforward graph traversal problem.

### 4. The badge layer is just substitution

`roadmap-badges.conf` does not contain a complicated cryptographic scheme.

It is simply:

```text
character -> hexadecimal value
```

So reversing it is trivial once the required badge values have been recovered.

### 5. Think like a parser

The key insight was:

> Don't attack the server. Attack the configuration.

By converting the Nginx configuration into structured data and walking the checkpoint graph backwards, the hidden route can be reconstructed deterministically.
