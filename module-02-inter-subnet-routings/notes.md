# Module 2 — Inter-Subnet Routing (Router VM, Forwarding, Gateways)

**Goal:** Prove that a "router" is nothing mystical — just a Linux machine with
two network interfaces (one per subnet) and IP forwarding enabled. Build two
separate subnets, connect them only through this router VM, and get a packet
to actually cross between them.

---

## 1. Topology

```
VM1 (192.168.10.10/24)                    VM2 (192.168.20.10/24)
   gateway 192.168.10.1                      gateway 192.168.20.1
        |                                          |
   [ lan10 network ]                        [ lan20 network ]
        |                                          |
        +---------------- Router VM ---------------+
          enp1s0: 192.168.10.1     enp7s0: 192.168.20.1
          (ip_forward = 1)
```

VM1 and VM2 are on **different** subnets this time (unlike Module 1, where
both shared one subnet). The router VM is the only device with a foot in
both networks.

---

## 2. What we built

- Two new isolated virt-manager networks: `lan10` (192.168.10.0/24) and
  `lan20` (192.168.20.0/24), replacing the shared `netlab` network from
  Module 1.
- VM1 moved onto `lan10`, VM2 moved onto `lan20` — reconfigured NICs and
  static IPs, one VM per subnet.
- A router VM created by **cloning** an existing VM (faster than a fresh
  install) and adding a second NIC, giving it:
  - `enp1s0` → `192.168.10.1/24` (leg into `lan10`)
  - `enp7s0` → `192.168.20.1/24` (leg into `lan20`)
- Static IPs + **gateway lines** added to VM1 and VM2's network config —
  new compared to Module 1, since now there's somewhere to route _to_.
- IP forwarding enabled on the router.
- Result (after a genuinely tricky debugging detour — see below): VM1 can
  ping VM2 across the router, with `ttl=63` confirming exactly one hop.

---

## 3. Key concepts, explained in depth

### 3.1 Why ARP alone isn't enough anymore

In Module 1, both devices shared one subnet, so ARP (a Layer 2, local-only
protocol) could resolve any MAC address directly — a broadcast physically
reaches every device on that segment. Here, VM1 and VM2 are on **different**
subnets, meaning different broadcast domains. **ARP cannot cross a subnet
boundary at all** — a switch/bridge will not forward an ARP broadcast from
one network into another. This is the fundamental reason routing has to
exist: without it, two devices on different subnets have no mechanism to
find each other's MAC addresses, no matter how "close" they physically are.

### 3.2 What a router actually is

Strip away the marketing: a router is a computer with **two or more network
interfaces**, each sitting on a different subnet, plus one specific
capability turned on — **IP forwarding**. By default, an OS refuses to
relay packets between its own interfaces; if something arrives on NIC A
destined for NIC B's subnet, the kernel just drops it, because a normal
host isn't supposed to forward other people's traffic. Flipping on IP
forwarding is the _entire_ technical difference between "a computer with
two network cards" and "a router." Everything else (NAT, firewalling, QoS,
dynamic routing protocols) is optional functionality layered on top of that
one core behavior.

### 3.3 The default gateway, explained precisely

Every outgoing packet triggers this check in the kernel: _"is the
destination IP inside my own subnet?"_

- **Yes** → same mechanism as Module 1: ARP for the destination directly,
  deliver via MAC.
- **No** → the kernel doesn't know how to reach that IP directly, so it
  hands the packet to whatever is configured as the **default gateway** —
  an IP address that _is_ on the sender's own subnet, so it can be ARPed
  for normally.

So "the gateway" isn't magic — it's simply the router's interface that
happens to sit on the sender's local segment. VM1 ARPs for the gateway's
MAC (ordinary Layer 2 resolution), then sends the packet to that MAC — but
the packet's **IP header still lists VM2 as the final destination**. The
router is only the next physical stop, not the final recipient. This is the
critical distinction from NAT (Module 3), where addresses in the packet
itself get rewritten — here, they never change.

### 3.4 The routing table and "next hop"

`ip route` output looks like:

```
default via 192.168.10.1 dev enp1s0
192.168.10.0/24 dev enp1s0 scope link
```

Each row answers: _"for packets going to this destination network, send
them out this interface, to this next-hop address."_ The `default` row is
the catch-all used when nothing more specific matches — exactly the row
consulted the moment a destination is recognized as non-local.

The router itself needs no "next hop" for its own two directly-attached
subnets — it just has two `scope link` entries, one per interface, saying
"this network is directly reachable right here."

### 3.5 The full packet journey, step by step

VM1 (`192.168.10.10`) pings VM2 (`192.168.20.10`):

1. VM1's kernel checks: is `192.168.20.10` local? No.
2. Consults its routing table → default gateway is `192.168.10.1`.
3. ARPs for `192.168.10.1` (this IS local — normal ARP applies).
4. Sends the ICMP packet, Ethernet-addressed to the router's MAC, but
   IP-addressed to `192.168.20.10`.
5. Router receives it on `enp1s0`. Because forwarding is enabled, it
   doesn't discard it — checks its own routing table.
6. Router sees `192.168.20.0/24` is directly attached via `enp7s0` →
   needs to forward it out there.
7. Router ARPs (separately, on the _second_ subnet) for `192.168.20.10`'s
   MAC.
8. Router sends the packet out `enp7s0` with a **new Ethernet header**
   (source = router's enp7s0 MAC, destination = VM2's MAC) — but the
   **IP header's source/destination addresses are untouched**. VM1 and
   VM2's IPs never change; only the Ethernet framing does, hop by hop.
9. VM2 receives it, replies, and the whole process runs in reverse.

This is literally longest-prefix-match and "next hop" at toy scale — the
same core logic that backbone routers run at internet scale, just with two
routing-table entries instead of hundreds of thousands.

---

## 4. The debugging journey — what actually held us back

Getting the topology built was the easy part. The ping then failed with
**"Destination Port Unreachable"** — and finding the real cause required
ruling out several plausible-sounding theories in order, using real
evidence at each step rather than guessing.

| #   | Suspected cause                                                                                                                                                                                                      | How we checked                                                                     | Result                                                                                                                                                         |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Active firewall rejecting forwarded traffic                                                                                                                                                                          | `nft list ruleset`, `iptables -L -n -v`, `systemctl status nftables`               | ❌ Ruled out — nothing active, `nftables.service` inactive                                                                                                     |
| 2   | Global IP forwarding not actually enabled                                                                                                                                                                            | `cat /proc/sys/net/ipv4/ip_forward`                                                | ❌ Ruled out — showed `1`                                                                                                                                      |
| 3   | **Per-interface forwarding disabled**, despite the global flag being on                                                                                                                                              | `cat /proc/sys/net/ipv4/conf/enp1s0/forwarding` and same for `enp7s0`              | ✅ Found `0` on both. Fixed with `sysctl -w net.ipv4.conf.<iface>.forwarding=1` — **but the ping still failed afterward, meaning this wasn't the whole story** |
| 4   | Reverse-path filtering (`rp_filter`) rejecting the traffic as looking asymmetric                                                                                                                                     | `cat /proc/sys/net/ipv4/conf/*/rp_filter`                                          | ❌ Ruled out — already in loose mode (`2`) on both interfaces                                                                                                  |
| 5   | **Duplicate IP address conflict** — the virt-manager network itself (the bridge, on the host) was ALSO configured with `192.168.10.1` / `192.168.20.1`, identical to what was manually assigned to the router's NICs | Checked `lan10`/`lan20`'s XML network definitions directly via `virsh net-dumpxml` | ✅ **This was the real root cause**                                                                                                                            |

### Why this was the actual problem

When virt-manager creates an "isolated network," it defaults to assigning
the bridge itself an IP address (so the bridge _could_ act as a gateway or
DHCP server if desired). This default wasn't needed here — the router VM
was meant to be the sole Layer 3 device on each subnet — but nobody removed
it. So two separate things were both claiming ownership of `192.168.10.1`
at the same time on the same Layer 2 segment: the host's bridge, and the
router VM's NIC. This silently corrupted ARP resolution and packet
delivery — traffic addressed to `.1` could be answered by either device
depending on timing, and packets could end up delivered to the host itself
(which has no knowledge of `192.168.20.0/24` or any forwarding rules)
instead of the router.

This is exactly why it took several wrong turns to find: every static
check on the _router itself_ (routing table, forwarding flags, rp_filter)
looked completely correct, because the router genuinely _was_ configured
correctly. The problem was a second, uninvited device competing for the
same address — invisible unless you specifically go looking at the virtual
network's own configuration, not just the VM's.

### The fix

```bash
# On the HOST machine (not inside any VM):
virsh -c qemu:///system net-edit lan10
# In the editor, delete the entire block:
#   <ip address="192.168.10.1" netmask="255.255.255.0">
#   </ip>
# Save and exit.

virsh -c qemu:///system net-edit lan20
# Same deletion for its <ip> block.

# Restart both networks to apply:
virsh -c qemu:///system net-destroy lan10 && virsh -c qemu:///system net-start lan10
virsh -c qemu:///system net-destroy lan20 && virsh -c qemu:///system net-start lan20

# Verify the <ip> block is gone:
virsh -c qemu:///system net-dumpxml lan10
virsh -c qemu:///system net-dumpxml lan20
```

After this, the router VM was the sole owner of `.1` on each subnet, and
the ping succeeded immediately — `0% packet loss`, `ttl=63` confirming
exactly one router hop.

---

## 5. Commands used

```bash
# --- Router: interface + forwarding setup ---
ip addr                                    # find interface names, confirm both NICs UP
sysctl -w net.ipv4.ip_forward=1            # global forwarding switch
cat /proc/sys/net/ipv4/ip_forward          # verify global switch = 1

# Per-interface forwarding (the subtlety that global ip_forward doesn't
# always cover)
cat /proc/sys/net/ipv4/conf/enp1s0/forwarding
cat /proc/sys/net/ipv4/conf/enp7s0/forwarding
sysctl -w net.ipv4.conf.enp1s0.forwarding=1
sysctl -w net.ipv4.conf.enp7s0.forwarding=1

# Reverse-path filtering check
cat /proc/sys/net/ipv4/conf/all/rp_filter
cat /proc/sys/net/ipv4/conf/enp1s0/rp_filter
cat /proc/sys/net/ipv4/conf/enp7s0/rp_filter

# Ask the kernel its actual routing decision for a destination
ip route get 192.168.20.10
# More precise: simulate the packet exactly as it arrives (source + incoming iface)
ip route get 192.168.20.10 from 192.168.10.10 iif enp1s0

# --- Firewall check (used to rule out REJECT rules) ---
nft list ruleset
iptables -L -n -v
systemctl status nftables

# --- VM1 / VM2: static IP + gateway config ---
# /etc/network/interfaces
auto enp1s0
iface enp1s0 inet static
    address 192.168.10.10
    netmask 255.255.255.0
    gateway 192.168.10.1      # new vs. Module 1 — tells the OS where
                                # to send anything that isn't local

systemctl restart networking

# --- Testing ---
ping -c 4 192.168.20.10
traceroute 192.168.20.10       # shows exactly one hop: the router

# --- Host machine: fixing the duplicate IP conflict ---
virsh -c qemu:///system net-list --all
virsh -c qemu:///system net-edit lan10
virsh -c qemu:///system net-edit lan20
virsh -c qemu:///system net-destroy lan10
virsh -c qemu:///system net-start lan10
virsh -c qemu:///system net-destroy lan20
virsh -c qemu:///system net-start lan20
virsh -c qemu:///system net-dumpxml lan10
virsh -c qemu:///system net-dumpxml lan20
```

---

## 6. Tools used

| Tool                                                                                  | Purpose                                                                                                                                                                            |
| ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `virt-manager` (KVM/QEMU)                                                             | Created `lan10`/`lan20` networks, cloned the router VM, added a second NIC                                                                                                         |
| `virsh` (`-c qemu:///system`)                                                         | Direct XML editing of network definitions — the GUI alone didn't expose the fix needed here                                                                                        |
| `ip` (iproute2)                                                                       | Interfaces, routing table, and `ip route get` for simulating routing decisions without sending real traffic                                                                        |
| `sysctl` / `/proc/sys/net/ipv4/...`                                                   | Reading and setting forwarding and rp_filter, both globally and per-interface                                                                                                      |
| `nft` / `iptables`                                                                    | Ruling out firewall rules as the cause                                                                                                                                             |
| **Wireshark**, capturing on both `virbr2` (lan10) and `virbr3` (lan20) simultaneously | The decisive diagnostic tool — showed exactly which bridge the packet reached (and didn't reach), proving the router wasn't forwarding at all rather than just routing incorrectly |
| `ping` / `traceroute`                                                                 | End-to-end connectivity and hop-count verification                                                                                                                                 |

---

## 7. What to remember vs. what to just look up

**Remember (durable concepts):**

- A router = 2+ NICs + IP forwarding enabled. That's the whole definition.
- ARP cannot cross subnet boundaries — this is _why_ routing exists at all.
- A gateway is just "the router's interface that happens to be on my local
  subnet" — nothing more mystical than that.
- **Forwarding is tracked per-interface, not only globally** — the global
  `net.ipv4.ip_forward` switch doesn't guarantee every individual interface
  has forwarding enabled.
- **Duplicate IP addresses on the same segment cause silent, confusing
  failures** that mimic completely different problems (looked like a
  forwarding/firewall issue for most of this debugging session). Worth
  checking early when static configuration all looks correct but traffic
  still fails.
- Debug outside-in: firewall → global forwarding → per-interface forwarding
  → rp_filter → addressing/conflicts — roughly in order of "most likely"
  before "most fundamental," but be ready to keep going past the first
  fix that seems plausible.

**Fine to look up when needed:**

- Exact `sysctl` / `/proc/sys/...` file paths
- Exact `virsh` command flags and syntax
- `vim` navigation keystrokes for quick edits
- Precise ICMP error code numbers/names

---

## 8. Recap — quiz yourself in a year

- **Q:** What's the minimal technical definition of a router?
  **A:** A machine with two or more network interfaces on different
  subnets, with IP forwarding enabled.

- **Q:** Why can't ARP resolve an address on a different subnet?
  **A:** ARP broadcasts don't cross subnet/broadcast-domain boundaries —
  a bridge/switch won't forward them into a different network segment.

- **Q:** Does the destination IP in a packet ever change as it passes
  through a router (without NAT)?
  **A:** No — only the Ethernet (MAC) framing changes at each hop; the
  IP header's source and destination stay the same end to end.

- **Q:** If the global `net.ipv4.ip_forward` shows `1`, is forwarding
  guaranteed to work on every interface?
  **A:** Not necessarily — check the per-interface value too:
  `/proc/sys/net/ipv4/conf/<iface>/forwarding`.

- **Q:** What was the actual root cause of the routing failure in this
  module, after ruling out firewall and forwarding issues?
  **A:** A duplicate IP address — the virt-manager network's own bridge
  was still holding the same `.1` address that had been manually assigned
  to the router VM's interface.

- **Q:** Why did this take several wrong turns to diagnose?
  **A:** Because every check performed _on the router itself_ (routing
  table, forwarding flags, rp_filter) was genuinely correct — the conflict
  lived in the virtual network's own configuration, a layer most
  debugging steps don't naturally look at first.
