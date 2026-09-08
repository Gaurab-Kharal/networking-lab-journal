# Module 1 — Same-Subnet LAN Communication (ARP & MAC Addressing)

**Goal:** Prove, hands-on, that two devices on the same subnet communicate
using only Layer 2 (MAC addressing via ARP) — with zero routing, zero NAT,
and zero gateway involvement. Everything abstract from earlier reading
(private IPs, broadcast domains, MAC resolution) becomes something you
built and watched happen in Wireshark.

---

## 1. What we built

- One **isolated** virtual network in virt-manager: name `netlab`, subnet
  `192.168.55.0/24`, backed by Linux bridge device `virbr1`. DHCP disabled,
  no forwarding to the host's physical NIC — meaning this network has no
  path to the internet or anything outside itself.
- Two minimal Debian VMs, `debian13(vm1)` and `debian13(vm2)`, both attached
  to `netlab` at creation time.
- Static IPs assigned by hand (not DHCP) on interface `enp1s0`:
  - VM1 → `192.168.55.10`
  - VM2 → `192.168.55.20`
- Result: `ping 192.168.55.20` from VM1 succeeds immediately, with no
  gateway configured anywhere.

---

## 2. Key concepts, explained in depth

### 2.1 Why no gateway was needed

A "gateway" (or default route) is only consulted when a device wants to
send a packet to an IP address **outside its own subnet**. The operating
system decides this by comparing the destination IP against its own
IP + subnet mask. If the destination falls inside the same subnet, the OS
knows the target is directly reachable on the local network segment — no
router needed, just a direct MAC-level delivery.

VM1 (`192.168.55.10/24`) and VM2 (`192.168.55.20/24`) are both within
`192.168.55.0/24`. So when VM1 pings VM2, the OS's very first internal
decision is: _"this is local, I don't need to ask a router, I just need
this device's MAC address."_ That's the entire reason routing never enters
the picture in this module — it's reserved for Module 2, where two
different subnets are introduced.

### 2.2 What ARP actually is, step by step

ARP (Address Resolution Protocol) exists because Ethernet hardware doesn't
know anything about IP addresses — it only understands MAC addresses. Every
frame sent on the wire (or virtual wire) needs a destination MAC, but
applications only ever specify a destination _IP_. ARP is the translation
layer that bridges that gap.

The exact sequence, matching what we captured in Wireshark:

1. VM1 wants to ping `192.168.55.20`. It checks its ARP cache
   (`ip neigh`) — no entry exists yet.
2. VM1 constructs an **ARP request** and sends it as an Ethernet frame
   with destination MAC `ff:ff:ff:ff:ff:ff` — the reserved Ethernet
   broadcast address. The payload asks, in effect, _"who has
   192.168.55.20? Tell 192.168.55.10."_
3. Because this is a broadcast, **every device connected to the same
   bridge (`virbr1`) receives a copy of this frame** — that's what
   broadcast means at Layer 2. In our two-VM lab that's just VM2, but on a
   larger LAN it would be every device on that segment.
4. Each device checks: _"is this IP mine?"_ Only VM2 recognizes
   `192.168.55.20` as its own address. Every other device (in a bigger LAN)
   silently discards the frame.
5. VM2 replies with an **ARP reply** — this time **unicast**, sent
   directly to VM1's MAC address (which VM2 learned from the request
   frame itself), saying _"192.168.55.20 is at 52:54:00:..."._
6. VM1 receives the reply and stores the IP→MAC mapping in its ARP cache.
7. **Only now** does the actual ICMP echo request (the "ping" itself) get
   sent — addressed directly to VM2's real MAC address, no more broadcasting
   needed.

This is exactly what packets 1–4 showed in the Wireshark capture: ARP
request → ARP reply → ICMP echo request → ICMP echo reply, in that order,
every single time a fresh mapping is needed.

### 2.3 Request vs. reply: broadcast vs. unicast

This distinction matters and is easy to blur:

|                 | Destination MAC                        | Who receives it             |
| --------------- | -------------------------------------- | --------------------------- |
| ARP **request** | `ff:ff:ff:ff:ff:ff` (broadcast)        | Every device on the segment |
| ARP **reply**   | The requester's specific MAC (unicast) | Only the original requester |

The request has to be broadcast because the sender doesn't yet know _who_
to ask directly — broadcasting is the only way to reach an unknown
recipient. The reply doesn't need to be broadcast because by that point the
replying device already knows exactly who asked (it read the source MAC
off the request frame).

### 2.4 ARP cache and the `STALE` state

Linux keeps a cache of IP→MAC mappings (view with `ip neigh` or the older
`arp -a`) so it doesn't have to re-run the ARP exchange for every single
packet — that would be wasteful. Entries naturally transition to `STALE`
after a timeout, which just means "not recently reconfirmed," not
"wrong." A stale entry is still used, but the kernel will trigger a fresh
ARP request in the background if it wants to re-verify the mapping (for
example, if a long gap in traffic passed, or before trusting it completely
again for new outbound traffic).

### 2.5 "Isolated" means no path _out_ — not no communication _within_

This tripped us up initially and is worth stating precisely: an isolated
virt-manager network has no bridge/route to the host's physical NIC and no
NAT — so nothing inside it can reach the internet, and nothing outside can
reach in. But that says **nothing** about communication between devices
_inside_ the isolated network. Two VMs on the same isolated subnet talk to
each other exactly as freely as if the network weren't isolated at all,
because same-subnet communication was never routed through the outside
world in the first place — it's pure local switching.

### 2.6 Why NAT never came up in this module

NAT (Network Address Translation) rewrites source/destination IP addresses
when traffic crosses between two _different_ address spaces — classically,
translating a private LAN IP into a public IP when leaving to the internet.
Since VM1 and VM2 share the exact same address space (`192.168.55.0/24`),
there is no boundary for NAT to operate across. NAT becomes relevant
starting in Module 3, once there's an "inside" and "outside" to translate
between.

### 2.7 Interface naming: `lo` vs `enp1s0`

- **`lo` (loopback)** is a purely software-defined interface — there's no
  physical hardware backing it on any machine, VM or not. It always
  resolves to `127.0.0.1` and represents "talk to yourself" — a program on
  a machine reaching another program on the _same_ machine through normal
  socket/networking APIs. This is why `localhost:5432` (a local Postgres
  connection, for example) works without any real network hardware
  involved at all.
- **`enp1s0`** follows systemd's predictable network interface naming
  scheme, which replaced the old `eth0`/`eth1` convention specifically
  because those numbers could silently shuffle between reboots depending on
  hardware detection order — a real source of production bugs historically.
  The name directly encodes physical (or virtual-as-physical) location:
  `en` = Ethernet, `p1` = PCI bus 1, `s0` = slot 0. In this lab, QEMU
  emulates a PCI Ethernet card for each VM, so the guest OS names it
  exactly as if it were real hardware sitting at that bus/slot.
- **`wlp...`** interfaces follow the same scheme but with `wl` for
  wireless — not something you'd normally see inside a bare VM unless a
  wireless device is explicitly passed through.

### 2.8 Why the host machine could see "isolated" traffic in Wireshark

The isolated network's bridge (`virbr1`) is a piece of software running
**on the host itself** — the host is not an outside party relative to this
network, it _is_ the switch implementing it. Every single frame exchanged
between VM1 and VM2 physically passes through that bridge device on the
host's kernel, which is exactly why running Wireshark on the host, capturing
on `virbr1` specifically, shows the full ARP + ICMP exchange in real time —
despite the VMs themselves having no route to the outside world at all.

---

## 3. What actually went wrong (and why it's worth keeping)

The debugging _is_ the part that made this stick — a clean walkthrough with
no errors would have taught far less.

1. **Forgot both VMs' login credentials** after a break between sessions.
   Recovered via GRUB rescue mode (`init=/bin/bash`).
2. **Typo #1:** wrote `init=bin/bash` — missing the leading slash. The
   kernel couldn't locate the binary at all, and the boot failed outright.
3. **Typo #2 (different VM):** wrote `/bin/bash` with no `init=` prefix.
   The kernel didn't recognize this as a valid parameter and silently
   ignored it, booting normally instead of dropping into rescue mode — no
   error shown, just unexpected normal behavior, which is arguably a
   trickier failure mode to notice than an outright crash.
4. **`passwd` failed** with "authentication token manipulation error" /
   "password unchanged." Root cause: the root filesystem was still mounted
   read-only (`ro`, part of the default rescue boot parameters) —
   `mount -o remount,rw /` was required before `passwd` could actually
   write to `/etc/shadow`.
5. **`sudo` didn't exist** on the minimal Debian netinst image at all, and
   root had no password set by default, so `su -` failed too. Resolved by
   setting a root password directly through rescue mode and using `su -`
   from then on, rather than trying to install `sudo` on a VM with no
   internet access (which wouldn't have worked anyway).

---

## 4. Commands used

```bash
# --- Inspection ---
ip addr                  # interfaces, IPs, link state
ip route                 # routing table
ip neigh                 # ARP/neighbor cache (modern)
arp -a                   # ARP cache (older/classic tool)

# --- Connectivity ---
ping 192.168.55.20
ping -c 4 192.168.55.20  # send exactly 4 pings and stop
ip neigh flush all       # clear ARP cache to force a fresh exchange

# --- Static IP config: /etc/network/interfaces ---
auto enp1s0
iface enp1s0 inet static
    address 192.168.55.10
    netmask 255.255.255.0

systemctl restart networking   # apply the config above

# --- GRUB rescue recovery ---
# (typed at the end of the "linux" boot line in GRUB's edit mode)
init=/bin/bash

# once dropped into the rescue shell:
mount -o remount,rw /
passwd root              # or: passwd <username>
cat /etc/passwd           # list accounts — real users have UID 1000+
exec /sbin/init            # resume normal boot without a full power cycle
```

---

## 5. Tools used

| Tool                                                   | What it was used for                                                                                                                        |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `virt-manager` (KVM/QEMU)                              | Created the VMs and the isolated `netlab` virtual network                                                                                   |
| `ip` (iproute2 suite)                                  | Inspected interfaces, IP assignments, routing table, and the ARP/neighbor cache                                                             |
| `nano`                                                 | Edited `/etc/network/interfaces`                                                                                                            |
| `ping`                                                 | Basic reachability test between VM1 and VM2                                                                                                 |
| **Wireshark** (run on the host, capturing on `virbr1`) | Captured the live ARP request/reply and ICMP exchange — proved the ARP mechanism directly instead of just trusting the textbook description |

---

## 6. Recap — quiz yourself in a year

- **Q:** Why didn't VM1 need a default gateway to reach VM2?
  **A:** Both are in the same subnet/broadcast domain — the OS recognizes
  the destination as local and resolves it via ARP directly, with no
  routing decision required.

- **Q:** What destination MAC does an ARP _request_ use, and why?
  **A:** `ff:ff:ff:ff:ff:ff` — the Ethernet broadcast address — because the
  sender doesn't yet know which device owns the target IP, so every device
  on the segment must receive the frame to check if it matches.

- **Q:** Is an ARP _reply_ broadcast too?
  **A:** No — it's unicast, sent directly to the original requester, since
  the replying device already knows exactly who to answer.

- **Q:** Why did Wireshark running on the host see traffic between two
  "isolated" VMs?
  **A:** Because the isolated network's bridge (`virbr1`) is software
  running on the host itself — the host isn't outside this network, it _is_
  the switch that implements it.

- **Q:** What does a `STALE` ARP entry mean?
  **A:** The mapping hasn't been recently reconfirmed — it's still usable,
  but the kernel may re-verify it with a fresh ARP request before fully
  trusting it again.

- **Q:** Why wasn't NAT involved anywhere in this module?
  **A:** NAT translates addresses across a boundary between two different
  address spaces. VM1 and VM2 share the exact same subnet — there's no
  boundary to translate across.
