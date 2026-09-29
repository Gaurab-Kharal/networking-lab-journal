# Module 3 — NAT: Source NAT (MASQUERADE) and Destination NAT (Port Forwarding)

**Goal:** Understand and build the two things NAT is actually used for in
practice: letting many private devices share one public address going
*out* (SNAT/MASQUERADE), and letting the outside world selectively reach a
specific private device *in* (DNAT/port forwarding) — the exact mechanism
behind a home router's "port forwarding" settings page.

This module builds directly on top of Module 2's router, rather than
starting fresh — same VMs, same two subnets, one new NIC added partway
through.

---

## 1. Why NAT exists at all

IPv4 has roughly 4.3 billion possible addresses — nowhere near enough for
every device on Earth. NAT is the practical workaround: many private
devices share a single public IP when talking outward. Your home router
does this right now for every device in your house.

## 2. The conceptual line between routing and NAT

This is the single most important distinction in this module:

- **Routing (Module 2)** never touches the addresses inside a packet — it
  only decides which interface to send it out. Source and destination IPs
  arrive at the far end exactly as they left.
- **NAT actively rewrites** the source or destination IP address in the
  packet header as it passes through, and keeps a **connection tracking
  table** so replies can find their way back to whichever internal device
  originally sent the request.

Two routed subnets (like Module 2's setup) never need NAT to talk to each
other — routing alone handles that, since both sides can see each other's
real IPs. NAT specifically matters when one side is genuinely *hidden* —
no route back to it exists at all from the other side.

---

## PART 1 — Source NAT (SNAT/MASQUERADE)

### 3. What we built

Reused the exact Module 2 topology (VM1 on `lan10`, VM2 on `lan20`, router
bridging both). Added one rule on the router: rewrite the source IP of
any traffic leaving via `enp7s0` (the `lan20`-facing interface) to the
router's own address on that interface.

### 4. The mechanism, step by step

1. VM1 (`192.168.10.10`) sends a packet toward VM2 (`192.168.20.10`).
2. It arrives at the router, routed exactly like Module 2.
3. Before forwarding it out `enp7s0`, the router **rewrites the source
   IP** to `192.168.20.1` (its own address on that interface) and records
   the mapping in its connection tracking table.
4. VM2 receives the packet — as far as VM2 can tell, the request came
   from `192.168.20.1`, not VM1. **VM2 never sees VM1's real IP at all.**
5. VM2's reply goes back to `192.168.20.1`. The router looks up its
   connection tracking table, recognizes this reply belongs to VM1's
   earlier request, and rewrites the destination back to `192.168.10.10`
   before forwarding it inward.

### 5. `MASQUERADE` vs. plain `SNAT`

`MASQUERADE` rewrites the source to whatever the outgoing interface's
*current* IP happens to be — the right choice when that IP might change
(a home router with a dynamic ISP-assigned address is the classic case).
Plain `SNAT` requires specifying a fixed IP and is slightly more efficient
when the address is static and known. We used `MASQUERADE` here.

### 6. A real obstacle: `iptables` isn't installed

Debian 13 (trixie) ships `nftables` as the default packet-filtering
backend. The `iptables` command isn't present on a minimal netinst install
at all, and since the router VM is isolated with no internet access,
installing it via `apt` wasn't an option either. `nft` was already present,
so the rule was built directly in native `nft` syntax instead — same
underlying framework, different command syntax.

### 7. Verifying it with Wireshark — the actual proof

Captured simultaneously on both `virbr2` (lan10) and `virbr3` (lan20),
filtered to `icmp`, then pinged from VM1 to VM2. Comparing the *same*
ping's packets across both bridges:

- On `virbr2` (inside): `192.168.10.10 → 192.168.20.10` — VM1's real IP,
  untouched.
- On `virbr3` (outside): `192.168.20.1 → 192.168.20.10` — the source has
  been rewritten to the router's address. The destination never changes,
  since VM2 was always the real target.

This is direct, visible proof that NAT is a rewrite operation, not a
routing decision.

---

## PART 2 — Destination NAT (Port forwarding)

### 8. What we built

```
HOST  <--->  [default network, host-reachable]  <---> Router (3rd NIC: enp8s0)  <---> lan20 (isolated) <---> VM2
```

Added a **third NIC** to the router, attached to virt-manager's built-in
`default` network (`192.168.122.0/24`) — the one network in this whole
setup that your actual host machine can reach directly. `lan20` (and
therefore VM2) remained fully isolated from the host, exactly as it's been
since Module 2. This is precisely the "hidden device behind a router"
scenario a real home network represents.

### 9. Confirming the "hidden" baseline first

Before adding anything, tested from the **host**:

```bash
curl 192.168.20.10:8000    # VM2's real address
```

This hung indefinitely (had to Ctrl+C) — no route exists at all from the
host to `lan20`.

```bash
curl 192.168.122.50:8000   # router's new default-network address
```

This failed **instantly** with "Connection refused" — the host *can*
reach the router, but nothing was listening on port 8000 there yet.

**This distinction matters and is worth remembering on its own:** a
*hang/timeout* means no route exists at all — the packet has nowhere to
go. An *instant refusal* means the destination is reachable, but nothing
is listening on that specific port. Reading a real connection failure and
immediately knowing which of these two situations you're in is a genuinely
transferable debugging skill.

### 10. The DNAT rule

Unlike MASQUERADE (which runs in `POSTROUTING`, after routing decisions
are made), a DNAT rule has to run in **`PREROUTING`** — *before* the
routing decision, so the kernel knows to route the packet toward VM2
rather than treating it as destined for the router itself.

```bash
nft add table ip nat2
nft add chain ip nat2 prerouting { type nat hook prerouting priority -100 \; }
nft add rule ip nat2 prerouting iifname "enp8s0" tcp dport 8000 dnat to 192.168.20.10:8000
```

Plain-language translation: *"any TCP traffic arriving on the `default`-network-facing interface, addressed to port 8000, gets its destination rewritten to VM2's real address and port before any routing decision is made."*

### 11. Proving it worked

With `python3 -m http.server 8000` running on VM2, from the **host**:

```bash
curl 192.168.122.50:8000
```

Returned a real HTML directory listing — served entirely by VM2, a machine
the host had no route to at all minutes earlier. The router transparently
relayed the connection through, exactly like a home router's port
forwarding configuration does for something like a home server or game
console.

---

## 12. The debugging journey — what actually went wrong

| # | Issue | What happened | Fix |
|---|---|---|---|
| 1 | Per-interface forwarding reset again | Same as Module 2 — `sysctl -w` settings don't survive a reboot, and rebooting the router (to add the 3rd NIC) wiped them | Re-applied, then finally made permanent via `/etc/sysctl.conf` for all three interfaces |
| 2 | `iptables` not installed | Debian 13 minimal netinst uses `nftables` by default; no internet to install `iptables` anyway | Used native `nft` syntax throughout instead |
| 3 | New NIC came up in `DOWN` state with no IP | Adding hardware doesn't automatically bring an interface up or request an address | `ip link set enp8s0 up`, then attempted DHCP |
| 4 | `dhclient` not installed | Same minimal-install/no-internet situation as `iptables` | Assigned a static IP manually instead: `ip addr add 192.168.122.50/24 dev enp8s0` |
| 5 | Part 1's MASQUERADE rule vanished | `nft` rules, like the forwarding sysctl, live only in kernel memory by default — the router reboot (for the 3rd NIC) wiped them too | Re-added the rule, then saved the full ruleset to `/etc/nftables.conf` and enabled the `nftables` service so it loads automatically at boot from now on |

**The throughline worth remembering:** almost everything in this module
that "broke" wasn't a conceptual misunderstanding — it was the same
non-persistence gotcha (settings living only in kernel memory, wiped on
reboot) showing up in three different places: forwarding, NAT rules, and
implicitly the missing tools. Recognizing "oh, this is that same class of
problem again" is itself a skill worth having.

---

## 13. What to actually remember (durable lessons)

- **NAT rewrites addresses; routing does not.** This is the single line
  that separates Module 2 from Module 3.
- **SNAT/MASQUERADE happens in POSTROUTING** (after routing decisions,
  as the packet is about to leave). **DNAT happens in PREROUTING**
  (before routing decisions, so the kernel knows where to actually send
  the now-rewritten packet).
- **A connection-tracking table is what makes NAT possible at all** —
  without it, a reply arriving at the router would have no way to know
  which internal device it belongs to.
- **Instant "connection refused" vs. a hang/timeout mean genuinely
  different things** — refused means reachable-but-nothing-listening;
  a hang means no route exists at all.
- **Kernel settings and firewall/NAT rules are not persistent by
  default** — anything set with `sysctl -w` or `nft add` needs to be
  explicitly written to a config file (`/etc/sysctl.conf`,
  `/etc/nftables.conf`) and the relevant service enabled, or it vanishes
  on the next reboot. This is now the second module in a row this exact
  lesson has shown up — treat it as a standing habit going forward, not a
  one-off fix.
- **A newly added network interface doesn't come up automatically** —
  it needs `ip link set <iface> up` and, without DHCP tooling available,
  a manually assigned static address.

## What's fine to just look up later

- Exact `nft` table/chain/rule syntax
- Exact `/etc/sysctl.conf` and `/etc/nftables.conf` file formats
- `systemctl enable` vs `systemctl start` distinction specifics

---

## 14. Commands used

```bash
# ===== PART 1: SNAT / MASQUERADE =====

# Re-confirm Module 2 routing still works before adding NAT
ping -c 4 192.168.20.10

# Check for iptables (not present on this system)
iptables -t nat -L -n -v      # command not found
which nft                      # /usr/sbin/nft — use this instead

# Build the MASQUERADE rule in native nft syntax
nft add table ip nat
nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }
nft add rule ip nat postrouting oifname "enp7s0" masquerade
nft list ruleset

# ===== PART 2: DNAT / Port forwarding =====

# Bring up the new 3rd NIC (added via virt-manager hardware settings)
ip link set enp8s0 up
dhclient enp8s0                                    # not installed — fell back to static
ip addr add 192.168.122.50/24 dev enp8s0
ip addr show enp8s0

# Confirm host reachability of the new interface (run on the HOST)
ping 192.168.122.50

# Confirm the "hidden" baseline (run on the HOST, before any DNAT rule)
curl 192.168.20.10:8000        # hangs — no route at all
curl 192.168.122.50:8000       # instant refusal — nothing listening yet

# Start the target web server on VM2
python3 -m http.server 8000

# Build the DNAT rule
nft add table ip nat2
nft add chain ip nat2 prerouting { type nat hook prerouting priority -100 \; }
nft add rule ip nat2 prerouting iifname "enp8s0" tcp dport 8000 dnat to 192.168.20.10:8000
nft list ruleset

# Confirm it worked (run on the HOST)
curl 192.168.122.50:8000       # now returns VM2's actual page

# ===== Making everything persistent =====

# Forwarding, for ALL interfaces (append to /etc/sysctl.conf)
echo "net.ipv4.conf.enp1s0.forwarding=1" >> /etc/sysctl.conf
echo "net.ipv4.conf.enp7s0.forwarding=1" >> /etc/sysctl.conf
echo "net.ipv4.conf.enp8s0.forwarding=1" >> /etc/sysctl.conf
sysctl -p

# nft rules — save current ruleset and enable auto-load at boot
nft list ruleset > /etc/nftables.conf
systemctl enable nftables
systemctl start nftables
nft list ruleset               # verify both tables (nat + nat2) survived
```

---

## 15. Tools used

| Tool | What it was used for |
|---|---|
| `nft` (nftables) | Built both the MASQUERADE (SNAT) and DNAT rules — the modern replacement for `iptables` on Debian 13 |
| `ip` (iproute2) | Brought the new NIC up, assigned it a static IP, inspected all interfaces |
| `curl` (on the host) | The actual test client proving both the "hidden" baseline and the working port-forward |
| `python3 -m http.server` | A zero-dependency target service on VM2 to make port-forwarding concretely testable |
| **Wireshark** (on the host, capturing `virbr2` + `virbr3` simultaneously) | Proved SNAT was a real rewrite operation by comparing the same packet's source IP on both sides of the router |
| `systemctl` | Enabled the `nftables` service so saved rules load automatically at boot |

---

## 16. Recap — quiz yourself in a year

- **Q:** What's the one-sentence difference between routing and NAT?
  **A:** Routing decides which interface a packet leaves through without
  touching its addresses; NAT actively rewrites the source or destination
  address as the packet passes through.

- **Q:** Why does MASQUERADE run in POSTROUTING but DNAT runs in
  PREROUTING?
  **A:** MASQUERADE needs to know the outgoing interface's IP, which is
  only known after the routing decision is made. DNAT needs to rewrite the
  destination *before* routing happens, so the kernel routes toward the
  new (real) destination.

- **Q:** VM2 replies to a NAT'd connection — how does the router know
  which internal machine that reply actually belongs to?
  **A:** The connection tracking table, built when the original outbound
  packet was rewritten — it records the mapping so replies can be
  correctly un-rewritten and delivered back to the right device.

- **Q:** A curl to a given IP:port hangs forever vs. fails instantly with
  "connection refused" — what's the difference in what's actually
  happening on the network?
  **A:** A hang means no route exists to that destination at all — the
  packet has nowhere to go. An instant refusal means the destination is
  reachable, but nothing is listening on that specific port.

- **Q:** Why did the router's forwarding settings and NAT rules disappear
  partway through this module?
  **A:** Both `sysctl -w` settings and `nft add` rules live only in
  kernel memory by default — rebooting the router (done here to add a
  third NIC) wiped both, requiring them to be explicitly saved to
  `/etc/sysctl.conf` and `/etc/nftables.conf` respectively to survive
  future reboots.

- **Q:** Why couldn't a routed-only setup (no NAT) let the host reach
  VM2 on `lan20`?
  **A:** The host has no route to `lan20` at all — routing only helps
  devices that already have a path between them via some chain of
  gateways. NAT/port-forwarding is specifically what makes a genuinely
  unreachable, hidden network segment selectively accessible.
