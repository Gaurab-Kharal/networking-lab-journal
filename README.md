# Networking Lab Journal

An evolving, hands-on record of learning networking from the ground up —
concept first, then a real hands-on lab to prove it, using Linux VMs
(`virt-manager`/KVM/QEMU) instead of certification-style memorization or
proprietary simulators.

**Approach, every module:** explain the concept in detail first (the _why_)
→ build a real lab that proves it → debug whatever actually breaks along
the way (this is usually where the real learning happens) → write it up
here in enough depth to be useful later.

**Environment:** Debian host, `virt-manager` (KVM/QEMU), minimal Debian
netinst VMs, Wireshark for packet-level verification.

---

## Status

| #   | Module                                                            | Status                                                     | Core concept                                                                                                    |
| --- | ----------------------------------------------------------------- | ---------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| 0   | Binary & subnetting                                               | ✅ Done conceptually — **not yet written up in this repo** | Subnetting is binary arithmetic in a costume; CIDR notation                                                     |
| 1   | [Same-subnet LAN](module-01-same-subnet-lan/README.md)            | ✅ Done                                                    | ARP + MAC addressing; no routing needed within one broadcast domain                                             |
| 2   | [Inter-subnet routing](module-02-inter-subnet-routings/README.md) | ✅ Done                                                    | A router is just a Linux box with two NICs and `ip_forward=1`; debugged a real duplicate-IP conflict end to end |
| 3   | NAT / iptables                                                    | ⬜ Not started                                             | Rewriting source IPs so many private hosts share one public IP                                                  |
| 4   | DNS                                                               | ⬜ Not started                                             | Name → IP resolution, happens _before_ routing, not instead of it                                               |
| 5   | Docker networking                                                 | ⬜ Not started                                             | Same virt-manager concepts, different backend (Docker bridge driver)                                            |
| 6   | Cloud VPC basics                                                  | ⬜ Not started                                             | Everything from Modules 0–3, as a managed cloud service                                                         |
| 7   | SSH tunnels                                                       | ⬜ Not started                                             | Reaching a host that isn't publicly exposed                                                                     |
| 8   | Kubernetes networking (optional)                                  | ⬜ Later                                                   | Pods = mini-LAN, Services = stable virtual IPs, Ingress = entry gateway                                         |

**Module 0 note:** the binary/subnetting math was worked through conceptually
before this repo existed, but was never written up as its own folder. It's
worth backfilling at some point — subnetting fluency is genuinely
load-bearing for reading almost any networking or cloud documentation, and
a short `module-00-binary-subnetting/README.md` with the core drills and a
couple of worked examples would round the journal out. Not urgent, but
flagged here so it doesn't get forgotten as more modules pile on top.

---

## How this repo is structured

```
networking-lab-journal/
├── README.md                              ← you are here (project-wide overview)
├── module-01-same-subnet-lan/
│   ├── notes.md                          ← everything for this module
│   └── images/
├── module-02-inter-subnet-routings/
│   ├── notes.md
│   └── images/
└── ...
```

**One `notes.md` per module** (not split into separate concepts/commands/
tools files) — GitHub auto-renders `notes.md` when browsing into a folder,
and keeping the story + reference material together in one file is easier
to maintain and easier to skim a year later than hopping between five tiny
files per module.

**Each module's `notes.md` follows the same shape:**

1. **What we built** — the concrete setup, plus a key screenshot
2. **Key concepts, explained in depth** — the _why_, written to actually be
   remembered, not just defined
3. **The debugging journey** — what actually broke and how it got diagnosed;
   often the most valuable section, since debugging under a real failure is
   what makes concepts stick, not reading a working config
4. **What to remember vs. what to look up** — an explicit split, since not
   everything deserves equal retention effort
5. **Commands used** — a reference block
6. **Tools used** — a short table
7. **Recap** — self-quiz Q&As for a fast refresher later

---

## The big-picture model tying every module together

Keep this translation table updated as new modules land — mapping homelab
vocabulary to real-world/cloud vocabulary is what makes later modules click
quickly instead of feeling like brand-new information:

| Homelab (virt-manager)                         | Real-world equivalent                                                                                                    |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Isolated `virt-manager` network (e.g. `lan10`) | An AWS/GCP VPC subnet                                                                                                    |
| `virbrX` bridge                                | A physical/virtual Ethernet switch                                                                                       |
| Router VM with two NICs + `ip_forward=1`       | A home router, or an AWS route table + Internet Gateway                                                                  |
| A duplicate IP address silently breaking ARP   | The kind of subtle networking bug that shows up in real cloud VPC misconfigurations too — same debugging instincts apply |
| `iptables` NAT rule _(Module 3, upcoming)_     | An AWS NAT Gateway                                                                                                       |

---

## How this project actually evolves

This isn't a fixed curriculum followed rigidly — it adapts based on what
comes up:

- Concepts get explained fully _before_ any lab work starts, every time.
- Labs are expected to break. Real debugging (checking one hypothesis at a
  time, ruling things out methodically) is treated as part of the
  curriculum, not a detour from it — Module 2's entire debugging journey is
  arguably more valuable than the "happy path" would have been.
- Modules get reused and extended rather than rebuilt from scratch each
  time (e.g. Module 2's router VM was cloned from a Module 1 VM rather than
  installed fresh) — this itself became a small lesson in what does and
  doesn't carry over when cloning a VM disk.
- Retention comes from re-use, not re-reading. Rebuilding a past lab
  slightly differently (add a VM, add NAT, add DNS) beats passively
  reviewing old notes.

---

## How to actually retain this (not just complete it once)

- **Subnetting math decays if unused.** Drill it occasionally even after
  Module 0 is "done" and written up.
- **Everything else, retention comes from re-use, not review.**
- **Always debug forward from a failure.** Every module so far has been
  more valuable because something broke first and got fixed through real
  troubleshooting.
