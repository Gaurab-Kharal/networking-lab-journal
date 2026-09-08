# Module 0 — Foundations: A Short History of Networking, the TCP/IP Model, and Subnetting

**Goal:** Build the conceptual bedrock everything else stands on. Before
touching a single VM, understand _why_ networks are shaped the way they
are, what a 32-bit IP address actually is, and how to subnet fluently by
hand — not by memorizing a calculator's output, but by understanding the
binary underneath it.

This module is unusual in that it's almost entirely conceptual — there's no
VM lab here. The "lab" is doing subnetting problems by hand until the
arithmetic feels boring rather than hard.

---

## 1. A short, practical history of networking

You don't need networking history trivia — but a little context makes the
design decisions in modern networking feel inevitable rather than
arbitrary.

- **1960s–70s: ARPANET.** The US Department of Defense funded a research
  network designed with one core requirement: it had to survive partial
  destruction (nuclear war was a real design constraint). This is why the
  internet has no single central point of control — it's a mesh of
  independently-operated networks, not one company's infrastructure. That
  single design choice explains an enormous amount of what follows: why
  routing exists, why there's no single "internet company," why an outage
  in one region doesn't take down the whole thing.
- **1970s–80s: TCP/IP is born.** Early networks used incompatible,
  proprietary protocols — a machine on one vendor's network often couldn't
  talk to a machine on another's. TCP/IP was designed specifically to be a
  common language that different networks could all agree to speak,
  regardless of what hardware or vendor was underneath. This is the
  origin of the layered model you'll use in every module of this
  curriculum.
- **1983: The "flag day."** ARPANET switched over to TCP/IP as its
  official protocol, essentially overnight. This is considered the
  technical birth of "the internet" as a concept — a network of networks,
  all agreeing to use the same addressing and routing rules.
- **1990s: The web (not the internet) is invented.** People often
  conflate these. The internet is the underlying network infrastructure
  (what this entire curriculum is about); the World Wide Web (HTTP, HTML,
  browsers) is just one application built on top of it, invented later by
  Tim Berners-Lee. Email, DNS, and file transfer all predate the web and
  run on the same underlying internet.
- **1990s–2000s: IPv4 address exhaustion becomes a real problem.** IPv4
  only has about 4.3 billion possible addresses — nowhere near enough for
  every device on Earth. Two major workarounds emerged: **NAT** (letting
  many private devices share one public address — Module 3) and **IPv6**
  (a vastly larger address space, designed to eventually replace IPv4
  entirely, though the transition is still ongoing decades later).
- **Today:** the same core protocols from the 1980s (IP, TCP, UDP) are
  still what everything runs on — cloud computing, Docker, Kubernetes, 5G,
  all of it is built on top of these same foundational rules, just with
  more layers of abstraction on top. This is _why_ this module matters:
  learn IP addressing and routing once, correctly, and it transfers to
  every layer built above it for the rest of your career.

---

## 2. What a network actually needs to solve

Strip away all jargon and every network — from two VMs to the entire
internet — has to answer exactly three questions:

1. **Addressing** — how do I identify _who_ a device is? (→ IP addresses)
2. **Delivery** — how does data actually get from one device to another
   across possibly many intermediate hops? (→ routing, switching, ARP)
3. **Reliability/ordering** — if data gets split into pieces to travel,
   how do the pieces get reassembled correctly, and what happens if some
   get lost? (→ TCP, and its simpler cousin UDP)

Every concept in this curriculum is really just a detailed answer to one
of these three questions.

---

## 3. The TCP/IP model, layer by layer

Real network traffic is organized into layers, each one only responsible
for its own job and blind to the layers above/below it. This separation is
deliberate — it's why you can swap out WiFi for Ethernet without changing
how your web browser works, or swap HTTP for a different application
protocol without needing new network cards.

| Layer           | Job                                         | Examples            |
| --------------- | ------------------------------------------- | ------------------- |
| **Application** | What the actual data _means_ to a program   | HTTP, DNS, SSH      |
| **Transport**   | Reliable (or not) delivery, ordering, ports | TCP, UDP            |
| **Internet**    | Addressing and routing across networks      | IP, ICMP            |
| **Link**        | Physical/local delivery within one segment  | Ethernet, ARP, WiFi |

**The "wrapping" mental model:** each layer takes what the layer above gave
it and wraps it in its own header, like nested envelopes. An HTTP request
gets wrapped in a TCP segment (adds ports + reliability info), which gets
wrapped in an IP packet (adds source/destination addresses), which gets
wrapped in an Ethernet frame (adds MAC addresses for local delivery). By
the time it hits the wire, it's envelopes inside envelopes inside
envelopes. This is exactly what you'll see when opening up any packet in
Wireshark — each layer visible as a nested section, outermost to innermost.

---

## 4. Binary and why IP addresses look the way they do

An IPv4 address is a **32-bit number**. Computers work in binary natively;
humans don't parse 32 raw bits easily, so IP addresses are written in
**dotted decimal** — four 8-bit chunks (octets), each converted to a
decimal number 0–255, separated by dots. `192.168.1.10` is really just
`11000000.10101000.00000001.00001010` written in a friendlier format.

**Why 0–255 per octet?** An 8-bit number can represent `2^8 = 256`
distinct values, counting from 0 — so 0 through 255.

**Binary place values**, the thing worth being able to do by hand without
a calculator:

```
Bit position:  128  64  32  16  8  4  2  1
Binary:          1   1   0   0  0 0  0  0
                128+64 = 192
```

Practice converting a few octets manually until this stops feeling like
math and starts feeling like reading.

---

## 5. Subnetting — the core skill of this module

### 5.1 What subnetting actually is

An IP address has two conceptual parts: a **network portion** and a
**host portion**. Subnetting is the act of deciding, out of the 32 total
bits, where that boundary falls. Everything else — CIDR notation, subnet
masks, "how many hosts fit" — is just consequences of that one decision.

Moving the boundary to allow more subnets means giving up host addresses
for that subnet, and vice versa. It's a direct trade-off, not two separate
resources.

### 5.2 CIDR notation and the subnet mask

`/24` means "the first 24 bits are the network portion." Written as a
mask, that's 24 consecutive `1` bits followed by `0`s:

```
/24  = 11111111.11111111.11111111.00000000  = 255.255.255.0
/25  = 11111111.11111111.11111111.10000000  = 255.255.255.128
/26  = 11111111.11111111.11111111.11000000  = 255.255.255.192
/27  = 11111111.11111111.11111111.11100000  = 255.255.255.224
/28  = 11111111.11111111.11111111.11110000  = 255.255.255.240
```

A `/27` mistake (writing 224 as something else) is a **notation slip**,
not a math slip, if the underlying binary is understood correctly — worth
remembering when debugging your own subnetting work later.

### 5.3 Working out host count from a mask

If `n` bits are used for the network portion, `32 - n` bits remain for
hosts. That gives `2^(32-n)` total addresses in the subnet — but **two are
always reserved** (the very first address = the network's identifier
itself, and the very last = the broadcast address for that subnet), so
usable hosts = `2^(32-n) - 2`.

### 5.4 The general method for a subnetting problem

Real-world subnetting problems usually give you a required number of
hosts and a required number of subnets, and ask for a mask. The reliable
method:

1. **Solve host bits first.** Find the smallest `h` such that
   `2^h - 2 >= required hosts`.
2. **Whatever bits remain determine subnet count.** With 16 usable bits in
   a typical class-B-sized block (as an example), subnet bits = `16 - h`,
   giving `2^(subnet bits)` possible subnets.

**Worked example** (a real problem solved during this module): _design for
30 subnets, 1510 hosts per subnet, starting from `172.19.0.0`._

- Host bits needed: find smallest `h` where `2^h - 2 >= 1510`.
  `2^11 - 2 = 2046 >= 1510` ✓ (2^10 - 2 = 1022, too small) → `h = 11`.
- Remaining bits for subnetting (out of the 16 host/subnet bits available
  in a `/16`-sized block): `16 - 11 = 5` → `2^5 = 32` subnets available
  (comfortably covers the required 30).
- Total network bits = `16 (original) + 5 (borrowed) = 21` → **`/21`**.
- `/21` as a mask: `255.255.248.0`.

This confirms the _method_ is understood (solve host bits from the
requirement first, then whatever bits remain determine subnet count) —
not just memorized for this one example. Any similar problem, different
numbers, same procedure.

### 5.5 Broadcast address vs. "next subnet start" — a common error

A specific mistake worth flagging because it's easy to make: the
**broadcast address** of a subnet (all host bits set to 1) is the _last_
usable-adjacent address in that subnet, reserved and unusable for a host.
It is NOT the same as the first address of the _next_ subnet, even though
they're numerically one apart. Example for `192.168.1.0/24`:

- Network address: `192.168.1.0` (unusable — identifies the subnet itself)
- First usable host: `192.168.1.1`
- Last usable host: `192.168.1.254`
- **Broadcast address: `192.168.1.255`** (unusable — sends to every host
  in _this_ subnet)
- Next subnet's network address: `192.168.2.0` (a completely different
  subnet, not a continuation)

---

## 6. Free resources to actually practice this

- **[subnetting.net](https://subnetting.net)** — timed subnetting drills,
  free, no account needed. This is genuinely the highest-value few hours
  in the entire curriculum: subnetting fluency is assumed background
  knowledge in nearly every networking or cloud document you'll ever read.
  20 minutes a day for a week is enough to build real fluency.
- **[Julia Evans' networking zines](https://wizardzines.com)** — cheap,
  extremely practical, illustrated, no fluff. Good for reinforcing
  concepts (like `iptables` later in Module 3) in a friendlier format than
  a textbook.
- **RFC 791 (IPv4)** and **RFC 950 (subnetting)** — the actual original
  specifications, if you ever want to read primary sources rather than
  secondhand explanations. Not necessary for fluency, but useful context
  that these aren't arbitrary rules — they were deliberately designed and
  documented decades ago.
- **`ipcalc`** (a command-line tool, install via `apt install ipcalc`) —
  useful for checking your manual subnetting work _after_ you've done it
  by hand, not as a substitute for doing it by hand.

---

## 7. What to actually remember vs. what's fine to look up

**Remember (this decays without practice — drill it periodically):**

- The general subnetting method (solve host bits first, then subnet bits)
- How to convert between CIDR notation, binary, and dotted-decimal masks
  for common values (`/24`, `/25`, `/26`, `/27`, `/28`)
- Network address and broadcast address are both reserved/unusable, and
  broadcast ≠ "start of next subnet"
- The TCP/IP four-layer model and the "wrapping" mental model

**Fine to look up when needed:**

- Exact `ipcalc` command syntax
- Precise historical dates (the concepts matter more than the trivia)
- IPv6 address format details (covered when directly relevant, later)

---

## 8. Recap — quiz yourself in a year

- **Q:** Why does the internet have no single central point of control?
  **A:** It descends from ARPANET, which was deliberately designed to
  survive partial destruction — a mesh of independently-operated networks
  rather than centralized infrastructure.

- **Q:** What's the difference between "the internet" and "the web"?
  **A:** The internet is the underlying network infrastructure (IP,
  routing, TCP). The web (HTTP, HTML, browsers) is one application
  built on top of it, invented later.

- **Q:** Why are IP addresses written in dotted decimal instead of binary?
  **A:** Purely for human convenience — computers work in the underlying
  32-bit binary natively; dotted decimal is just a friendlier notation for
  people to read and write.

- **Q:** If you need 1510 hosts per subnet, how many host bits do you need,
  and why?
  **A:** 11 bits — because `2^11 - 2 = 2046 >= 1510`, while `2^10 - 2 =
1022` is too small. The `-2` accounts for the reserved network and
  broadcast addresses.

- **Q:** Is a subnet's broadcast address the same as the next subnet's
  starting address?
  **A:** No — they're numerically adjacent but conceptually completely
  different. The broadcast address is reserved and unusable within its
  own subnet; the next subnet's network address belongs to an entirely
  separate subnet.
