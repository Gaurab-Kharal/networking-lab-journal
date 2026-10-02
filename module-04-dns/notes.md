# Module 4: DNS

DNS is the phonebook that turns names into IP addresses **before** anything else happens. Addressing, ARP, routing and NAT (Modules 1–3) only ever deal in IPs. DNS is the step that happens first, when you type a name.

**Tag legend** (used throughout):

- **[CONCEPT]** is true on any network or cloud. Remember it.
- **[LINUX]** is how Linux does it (files, commands). Learn once, reuse on any Linux box.
- **[TOOL]** is one program's own syntax (`dig`, `dnsmasq`). Look it up when needed.
- **[LAB]** is a setup chore or detail of this VM lab only. Don't memorize it.

---

## 1. What we built

A small DNS server for the lab, running on the router VM.

- **dnsmasq** on the router, listening on `192.168.10.1` (lan10) and `192.168.20.1` (lan20) only, not on the `default`-network side.
- Three local names, all under `.lab`:
  - `vm1.lab` → `192.168.10.10`
  - `vm2.lab` → `192.168.20.10`
  - `router.lab` → `192.168.10.1` and `192.168.20.1` (two records)
- VM1's `/etc/resolv.conf` points at `192.168.10.1`, so `ping vm2.lab` works through real DNS.
- Then we tested it by breaking it on purpose, and watched one lookup as packets in Wireshark.

The dnsmasq config used (`/etc/dnsmasq.conf`):

```
interface=enp1s0
interface=enp7s0
bind-interfaces
local=/lab/
domain=lab
host-record=vm1.lab,192.168.10.10
host-record=vm2.lab,192.168.20.10
host-record=router.lab,192.168.10.1
host-record=router.lab,192.168.20.1
```

### Setup chores (not DNS, just getting the tool installed) [LAB]

The VMs have no internet, so dnsmasq could not be installed normally.

1. The router's third NIC (`enp8s0`) sits on virt-manager's `default` network, which the host NATs to the internet. After the router lost its manual settings, we re-did them: `ip link set enp8s0 up`, `ip addr add 192.168.122.50/24 dev enp8s0`, `ip route add default via 192.168.122.1`, and `nameserver 192.168.122.1` in `/etc/resolv.conf`.
2. `apt update` failed because the only apt source was the install ISO (`cdrom:`). We replaced `/etc/apt/sources.list` with the online Debian repository.
3. `apt install dnsmasq dnsutils` (the second package provides `dig`).

None of this is about DNS. It is the cost of working in a lab with no internet.

---

## 2. Key concepts in depth

### 2.1 A name is resolved first, then packets flow [CONCEPT]

`ping vm2.lab` does two separate jobs:

1. Turn the name into an IP (DNS, or a local file).
2. Only then send the actual packets, using the normal path from Modules 1–3.

If step 1 fails, **no ping packet is ever sent.** If you ping the IP directly you skip step 1 and only need ARP, routing and forwarding. We proved this: with dnsmasq stopped, `ping <name>` failed but `ping 192.168.20.10` still worked.

So **"it fails by name but works by IP" means the problem is DNS, not the network.** Testing by IP first, then by name, separates the two.

### 2.2 Two places to look, in order [CONCEPT] [LINUX]

A machine looks for a name in:

1. **`/etc/hosts`**: a plain text file of `IP name` pairs on that machine only.
2. **A DNS server**, whichever one `/etc/resolv.conf` names.

The local file wins. We proved it: with `10.99.99.99 vm2.lab` in `/etc/hosts`, `ping vm2.lab` went to `10.99.99.99` even though dnsmasq would have said `192.168.20.10`. A forgotten hosts entry silently overrides real DNS. This is a classic real-world trap.

- **[LINUX]** `/etc/resolv.conf` says *which server to ask*. `nameserver 192.168.10.1` is the whole file for VM1.
- **[LINUX]** The lookup order is set by the `hosts:` line of `/etc/nsswitch.conf`. We did not view that file in this lab (not verified here).
- **[TOOL]** `getent hosts <name>` asks the way normal programs do, following that order. It is part of the base system, so it exists on every Linux box.

### 2.3 What a lookup looks like on the wire [CONCEPT]

A DNS lookup is one small question and one small answer, carried in UDP packets on **port 53**. From the capture on lan10's bridge (`virbr2`):

| Packet | From → To | What it is |
|---|---|---|
| 165 | `192.168.10.10:60968` → `192.168.10.1:53` | Query `0xf773`: type **A** for `vm2.lab` |
| 166 | `192.168.10.10` → `192.168.10.1:53` | Query `0x0d8d`: type **AAAA** for `vm2.lab` |
| 167 | `192.168.10.1:53` → `192.168.10.10` | Reply `0xf773`: **A 192.168.20.10** |
| 168 | `192.168.10.1:53` → `192.168.10.10` | Reply `0x0d8d`: no address |

What this shows:

- **The question comes from a random high port to port 53**, and the reply comes back from port 53 to that high port.
- **The transaction ID pairs each answer with its question.** Two questions were in flight at once, so the ID is how VM1 knows which reply is which. Wireshark shows it as `[Response In: 167]`.
- **A and AAAA are asked together.** Modern resolvers ask for the IPv4 and IPv6 address at once. We only defined IPv4 records, so the AAAA reply carries no answer.
- **It is fast.** The A reply arrived about 1.15 ms after the query.
- **Layers wrap:** Ethernet 14 + IP 20 + UDP 8 + DNS 25 = 67 bytes, exactly the length of the query frame. This is the Module 0 "wrapping" idea with DNS as the top layer.
- **The Ethernet destination was the router's own MAC**, because VM1 and the router share lan10. VM1 reached its DNS server by ARP and MAC (Module 1), with no gateway involved.

### 2.4 Record types [CONCEPT]

| Type | Meaning |
|---|---|
| **A** | name → IPv4 address (the main one) |
| **AAAA** | name → IPv6 address |
| **CNAME** | alias pointing to another name (we saw it: `deb.debian.org` is an alias for `debian.map.fastlydns.net`) |
| **PTR** | reverse lookup, IP → name |

`ping` printed `router.lab (192.168.10.1)` in one output, which looks like a reverse lookup of that address. We did not capture or confirm that, so treat it as likely, not proven.

### 2.5 The four ways a lookup can end [CONCEPT]

| Outcome | What it means | What to check |
|---|---|---|
| **An address** | It worked. | Nothing. |
| **NXDOMAIN** | A server answered: "that name does not exist." | Typo, or missing record. |
| **REFUSED** | A server got the question but won't answer it. | The server's own settings or upstream. |
| **Silence** | Nobody answered, so the client waits and retries. | `resolv.conf`, is the server up, is there a route. |

The speed is part of the clue. **An answer, even a negative one, is fast. Only silence is slow**, because the asker cannot tell "nobody there" from "reply is just late", so it waits and retries before giving up. This is the same split as "refused vs hang" in Module 3.

### 2.6 Resolver timing [LINUX]

With `resolv.conf` pointing at an address nobody owns (`192.168.10.99`), the lookup failed after **11.66 seconds** with `Temporary failure in name resolution`. As far as I know the default is about 5 seconds per attempt and 2 attempts, which is close. The extra ~1.7 s is unexplained.

The message `Temporary failure in name resolution` means "I could not get any usable answer." It does not say why. The clock helps tell the cases apart.

### 2.7 NXDOMAIN from the server side [CONCEPT] [TOOL]

`dig @192.168.10.1 nothing.lab` returned `status: NXDOMAIN`, `ANSWER: 0` and `Query time: 0 msec`. The server answered instantly that the name does not exist.

- The flags were `qr rd ra`. The earlier `vm1.lab` answer also had `aa`. We did not work out why `aa` is missing on the NXDOMAIN reply, so do not rely on the `aa` flag. The `status:` word is the reliable part.
- `local=/lab/` makes dnsmasq answer `.lab` names itself and never forward unknown `.lab` names upstream. That is why the "no" comes back immediately.

### 2.8 Authoritative vs recursive [CONCEPT]

Not done in the lab, but part of the picture:

- An **authoritative** server holds the real records for a zone. In the lab, dnsmasq was the authority for `.lab`.
- A **recursive resolver** does the legwork for you: it asks the root servers, then `.com`, then the domain's own servers, and caches the result. `dig +trace` shows that chain (not run here).

### 2.9 Why this matters for data engineering [CONCEPT]

- Cloud endpoints (RDS, Kafka brokers, Redshift) are hostnames, often CNAMEs whose IPs can change.
- Docker's built-in DNS (Module 5) lets containers find each other by name. It is the same idea as this module.
- "Works from my laptop, fails from the private subnet" is often DNS, not routing.
- Kubernetes service discovery is DNS.

---

## 3. Debugging journey

| What happened | How it was checked | What it taught | Tag |
|---|---|---|---|
| `vm2.lab` returned `NXDOMAIN` | `dig` header: `status: NXDOMAIN` | A server answered, so it is a config problem. Config had `vm1.lab` on two lines and no `vm2.lab`. | [TOOL] |
| `router.lab` returned only one address | `dig +short`, then `cat /etc/dnsmasq.conf` | One `host-record` line holds one IPv4 address. To give a name two addresses, repeat the name on separate lines. | [TOOL] |
| `dig @192.168.10,1 ...` failed with "couldn't get address" | Read the error text | A comma instead of a dot made `dig` treat it as a *hostname* and try to resolve it first. The real server was never contacted. | [TOOL] |
| `sed -i` said "no input files" | Read the error text | `sed -i` edits a file in place, so it needs the filename at the end. | [LINUX] |
| A step was run on the router instead of VM1 | Prompt said `root@router`, and `ttl=64` instead of `63` | Always check which machine you are on. TTL 63 means one router in between. | [CONCEPT] |
| "The lab is broken" | Bottom-up check on router and VM1 | Every layer passed: interfaces, forwarding, dnsmasq listening, gateway, route, name. Nothing was broken. | [CONCEPT] |
| `enp8s0` was `DOWN` with no address and no default route | `ip -br addr`, `ip route` | Expected: those were set by hand and not saved. It only affects internet access, not lab DNS. | [LAB] |
| `dig: command not found` on VM1 | Read the error | `dig` was only installed on the router, which had internet. Use `getent hosts` on the VMs. | [LAB] |
| Wireshark showed hundreds of DNS lines | Read the names in the Info column | They were lookups for `1.debian.pool.ntp.org` and similar, answered `Refused`, from VM1's background time sync. Not our test. | [TOOL] |
| `apt update` failed on a `cdrom:` source | Read the error | "Network works but apt fails" usually means a bad source entry. | [LAB] |

### Bottom-up troubleshooting [CONCEPT]

Look before you change anything, and start at the lowest layer. A broken lower layer makes everything above it look broken. The order we used:

1. Is the interface up with the right IP? (`ip -br addr`)
2. Is there a route and a default gateway? (`ip route`)
3. Is forwarding on for each router interface? (`cat /proc/sys/net/ipv4/conf/<iface>/forwarding`)
4. Is the DNS service running **and** listening where it should? (`systemctl is-active`, `ss -ulnp | grep :53`). "Started" and "listening on the right address" are different facts.
5. Can I reach things by IP? Then by name?

---

## 4. Experiments and results

| Test | Result |
|---|---|
| Name in `/etc/hosts` only, no DNS server | Resolved, `ping` worked, `ttl=63` |
| `resolv.conf` → router's dnsmasq, hosts line removed | Resolved over the network, `ttl=63` |
| dnsmasq stopped, ping by name | Name lookup failed. Time and message: **not measured** |
| dnsmasq stopped, ping by IP | Worked (skips DNS) |
| `resolv.conf` → `192.168.10.99` (nobody there) | 11.66 s, `Temporary failure in name resolution` |
| `nothing.lab` (server side) | `dig`: `NXDOMAIN`, `0 msec` |
| `nothing.lab` (VM1 side, time and `ping` message) | **not measured** |
| Hosts entry `10.99.99.99 vm2.lab` | Hosts file won. `ping` used `10.99.99.99` and got `Destination Net Unreachable` from the router |
| Does `dig` ignore the hosts file? | **not measured** (`dig` not installed on VM1) |
| DNS in Wireshark | 4 packets: A and AAAA queries, matching replies, IDs paired |

A stale hosts entry gives a different symptom than a DNS failure: the lookup **succeeds**, the connection **fails**.

---

## 5. What to remember vs what to look up

### Remember [CONCEPT]

- A name is resolved first; if that fails, no packet is sent.
- Order: `/etc/hosts` first, then the DNS server in `/etc/resolv.conf`.
- A lookup is a UDP question and answer on port 53. The transaction ID pairs them.
- Record types: A, AAAA, CNAME, PTR.
- NXDOMAIN means the server answered "no such name". Silence means nobody answered, and it is slow.
- Test by IP first, then by name. That separates network faults from DNS faults.
- "Started" is not the same as "listening in the right place".

### Look up when needed [TOOL] [LINUX]

- dnsmasq config lines (`host-record=`, `local=`, `interface=`, `bind-interfaces`, `domain=`).
- `dig` options and how to read its output (read `status:` and the ANSWER SECTION first).
- `sed`, `apt`, `systemctl` syntax.
- Interface names and bridge names.

### Do not memorize [LAB]

- The temporary internet steps and the `cdrom` fix.
- The names `vm1.lab`, `vm2.lab`, `router.lab`. `.lab` is not special.
- In the real world a data engineer almost never runs dnsmasq. The cloud provides DNS. The takeaway is the concepts.

---

## 6. Commands used

```bash
# --- Local names without any DNS server ---
cat /etc/hosts
echo "192.168.20.10 vm2.lab" >> /etc/hosts      # add a local name
sed -i '/vm2.lab/d' /etc/hosts                    # remove it again
getent hosts vm2.lab                              # resolve the way programs do
grep vm2 /etc/hosts                               # check for a leftover entry

# --- Which DNS server does this machine ask? ---
cat /etc/resolv.conf
echo "nameserver 192.168.10.1" > /etc/resolv.conf
cp /etc/resolv.conf /etc/resolv.conf.bak          # back up before breaking it

# --- dnsmasq (on the router) ---
dnsmasq --test                                    # check config syntax
systemctl restart dnsmasq
systemctl status dnsmasq
systemctl is-active dnsmasq
systemctl stop dnsmasq                            # used to break it on purpose
systemctl start dnsmasq
ss -ulnp | grep :53                               # who is listening on UDP 53, and where

# --- Asking a DNS server directly (on the router) ---
dig @192.168.10.1 vm2.lab
dig @192.168.10.1 vm2.lab +short                  # only the answers

# --- Timing a failure ---
time getent hosts vm2.lab

# --- Health checks ---
ip -br addr
ip route
cat /proc/sys/net/ipv4/conf/enp1s0/forwarding
ping -c1 192.168.10.1
ping -c1 192.168.20.10

# --- Setup chores (router, temporary internet) ---
ip link set enp8s0 up
ip addr add 192.168.122.50/24 dev enp8s0
ip route add default via 192.168.122.1
apt install dnsmasq dnsutils
hostnamectl set-hostname router                   # rename a cloned machine

# --- Wireshark (host) ---
virsh -c qemu:///system net-info lan10            # shows Bridge: virbr2
#   display filter: dns
#   display filter: dns.qry.name == "vm2.lab"
```

---

## 7. Tools used

| Tool | What it did here | Tag |
|---|---|---|
| `dnsmasq` | The DNS server for `.lab` names | [TOOL] |
| `dig` (from `dnsutils`) | Asks a server a question and shows the full reply | [TOOL] |
| `getent hosts` | Resolves a name the way normal programs do | [LINUX] |
| `/etc/hosts`, `/etc/resolv.conf` | Local name list, and which server to ask | [LINUX] |
| `systemctl`, `ss`, `ip` | Check the service, what is listening, and the interfaces | [LINUX] |
| `time` | Measures how long a failure takes | [LINUX] |
| Wireshark | Shows the DNS packets on lan10's bridge | [TOOL] |
| `virsh -c qemu:///system` | Finds the bridge name (`virbr2`) for lan10 | [TOOL] |

---

## 8. Recap quiz

**Q1. Why does `ping 192.168.20.10` work while `ping vm2.lab` fails when dnsmasq is stopped?**
An IP skips name resolution and goes straight to ARP, routing and forwarding. A name needs DNS to answer first.

**Q2. What is the order of lookup on a Linux machine?**
`/etc/hosts` first, then the DNS server named in `/etc/resolv.conf`.

**Q3. Why did the lookup take 11.66 seconds with a wrong nameserver, but `dig` for `nothing.lab` took 0 ms?**
Nobody answered at the wrong address, so the client waited and retried. The real server answered "no such name" immediately. An answer is fast, silence is slow.

**Q4. What does NXDOMAIN mean, and what would you check?**
The server answered that the name does not exist. Check for a typo or a missing record, not the network.

**Q5. Why did `ping vm2.lab` go to `10.99.99.99`?**
The local `/etc/hosts` is checked before DNS, so the hosts entry won.

**Q6. In the Wireshark capture, how does VM1 know which reply belongs to which question?**
Each pair shares a transaction ID (`0xf773` for the A query and reply, `0x0d8d` for the AAAA pair).

**Q7. Why were there two queries for one `ping vm2.lab`?**
The resolver asked for an A record (IPv4) and an AAAA record (IPv6) at the same time. We only defined IPv4, so the AAAA reply had no address.

**Q8. dnsmasq is `active` but nothing resolves. What is the next check?**
`ss -ulnp | grep :53`. The service can be running but not listening on the address the clients use.

**Q9. Why did the stale hosts entry give `Destination Net Unreachable` instead of a name error?**
The name resolved fine, to a wrong address. Then the router had no route to `10.99.99.99` and said so. The lookup succeeded and the connection failed.

---

## 9. Not measured

These were planned but not measured, so no results are recorded:

- Time and `ping` error message with dnsmasq stopped.
- VM1-side time and `ping` message for the non-existent name `nothing.lab`.
- Whether `dig` ignores `/etc/hosts` (`dig` is not installed on VM1).
- The contents of `/etc/nsswitch.conf`.
- Whether `ping` printed `router.lab` through a reverse (PTR) lookup.
