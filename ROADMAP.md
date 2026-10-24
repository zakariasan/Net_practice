# Networking Roadmap — from zero to BGP

> The [README](README.md) is the deep reference. **This file is the path**: what to learn first, what to watch, what to practise, and how to know you're ready for the next step.

Don't read everything at once. Take **one stage at a time**, watch the videos, do the exercise, and pass the checkpoint before moving on. About 30–60 minutes a day is enough.

```
 FOUNDATIONS                    CORE                        APPLIED                    ADVANCED
 ───────────                    ────                        ───────                    ────────
 1 What is a network      ──►   4 Routing            ──►    6 TCP, UDP, apps    ──►    8 Switching deeper (VLAN, STP)
 2 Binary & IP addresses        5 NAT & the Internet        7 Network programming      9 Dynamic routing (OSPF, BGP)
 3 Subnetting                   ✔ NetPractice 5–10          ✔ webserv / ft_irc        10 IPv6
 ✔ NetPractice 1–4                                                                    11 Security
                                                                                      12 Cloud & modern networks
```

---

## How to use each stage

Every stage has the same blocks:

- **Goal:** what you'll be able to do.
- **Learn:** the ideas, in order.
- **Watch / Read:** resources, best first.
- **Do:** hands-on practice.
- **Checkpoint:** if you can do this without notes, move on.
- **Deeper:** where it's explained in the README.

---

# Foundations

## Stage 1 — What is a network?

**Goal:** understand the words before the maths.

**Learn**
1. A network = devices that exchange data: hosts (PC, phone, server), switches, routers.
2. Client vs server: who asks, who answers.
3. Every device needs an **address**: a MAC (hardware, local) and an IP (logical, global).
4. A **port** says which program on the device gets the data (web = 80/443).
5. The Internet = millions of networks joined by routers.

**Watch**
- NetworkChuck, *You SUCK at Subnetting*, episode 1 "What is an IP address?": [playlist](https://www.youtube.com/playlist?list=PLIhvC56v63IKrRHh3gvZZBAGvsvOhwrRF)
- Practical Networking, *Networking Fundamentals*, first videos: [playlist](https://www.youtube.com/playlist?list=PLIFyRwBY_4bRLmKfP1KnZA6rZbRHtxmXi)

**Do**
```bash
ip addr        # find your IP and mask
ip route       # find your default gateway (your home router)
ping 8.8.8.8   # can you reach the Internet?
```

**Checkpoint:** explain to a friend what a host, a switch, a router, an IP and a port are, in one sentence each.

**Deeper:** [README §1](README.md#1-the-big-picture-osi-and-tcpip)

---

## Stage 2 — Binary and IP addresses

**Goal:** read any IP address and mask, and see the bits behind them.

**Learn**
1. Binary: each octet = 8 bits. The bit values are `128 64 32 16 8 4 2 1`.
2. Convert decimal ↔ binary: `192 = 11000000`, `255 = 11111111`, `224 = 11100000`.
3. An IPv4 address = 4 octets = 32 bits.
4. The mask: `1`s = network part, `0`s = host part. `255.255.255.0` = `/24`.
5. Special addresses: `127.x` (loopback), private ranges (`10/8`, `172.16/12`, `192.168/16`).

**Watch**
- NetworkChuck, *You SUCK at Subnetting*, episodes on binary and subnet masks: [playlist](https://www.youtube.com/playlist?list=PLIhvC56v63IKrRHh3gvZZBAGvsvOhwrRF)

**Do**
- Convert 10 random numbers 0–255 to binary and back, by hand.
- Write these masks in both forms: /8, /16, /24, /25, /26, /27, /28, /30.

**Checkpoint:** you can write `255.255.255.192` as `/26` and as `11111111.11111111.11111111.11000000` without help.

**Deeper:** [README §5](README.md#5-layer-3-ip-addressing-and-subnetting)

---

## Stage 3 — Subnetting

**Goal:** given any IP and mask, find the network, broadcast and host range in under a minute.

**Learn**
1. Block size = `256 − mask octet` (e.g. /27 → 256 − 224 = 32).
2. Network = largest multiple of the block ≤ the address.
3. Broadcast = network + block − 1. Usable = everything in between.
4. Lower `/n` = bigger network (each −1 doubles it).
5. Splitting a block into smaller subnets (VLSM): biggest first, blocks start on multiples of their size.

**Watch: Practical Networking, Subnetting Mastery** (in order, the best subnetting course on YouTube)
| Part | Video |
| --- | --- |
| 1 | [watch](https://www.youtube.com/watch?v=BWZ-MHIhqjM) |
| 2 | [watch](https://www.youtube.com/watch?v=ljS07YTEJ2I) |
| 3 | [watch](https://www.youtube.com/watch?v=5-wlfAdcmFQ) |
| 4 | [watch](https://www.youtube.com/watch?v=SM0kdVfhxZ0) |
| 5 | [watch](https://www.youtube.com/watch?v=Pj_ubhpi0WQ) |
| 6 | [watch](https://www.youtube.com/watch?v=p-6BVEbb9q0) |
| 7 | [watch](https://www.youtube.com/watch?v=OQ-r_IfeB8c) |

Full playlist: [here](https://www.youtube.com/playlist?list=PLIFyRwBY_4bQUE4IB5c4VPRyDoLgOdExE) · written version: [Subnetting Mastery article](https://www.practicalnetworking.net/stand-alone/subnetting-mastery/)

Then the rest of NetworkChuck's series (private IPs, NAT, VLSM): [playlist](https://www.youtube.com/playlist?list=PLIhvC56v63IKrRHh3gvZZBAGvsvOhwrRF)

**Do**
- 20 problems a day on [subnettingpractice.com](https://subnettingpractice.com/) for a week.
- Check your answers with `ipcalc 77.79.17.3/25`.
- Split `192.168.1.0/24` into: one /25, one /26, two /27.

**Checkpoint:** ✔ **NetPractice levels 1–4** solved without help.

**Deeper:** [README §5](README.md#5-layer-3-ip-addressing-and-subnetting), [README levels 1–4](README.md#13-levels-110)

---

# Core

## Stage 4 — How packets move: models, switches, ARP, routers

**Goal:** follow a packet hop by hop and say what every device does with it.

**Learn**
1. The OSI model (7 layers) and the TCP/IP model (4 layers).
2. Encapsulation: data → segment → packet → frame → bits.
3. Switch = layer 2 (MAC), one network. Router = layer 3 (IP), joins networks.
4. ARP: how a host finds the MAC of its gateway.
5. Packet vs frame: the IP stays the same end to end, the MACs change at every hop.

**Watch / Read**
- Practical Networking, *Networking Fundamentals*: [playlist](https://www.youtube.com/playlist?list=PLIFyRwBY_4bRLmKfP1KnZA6rZbRHtxmXi)
- *Packet Traveling* articles, in order:
  [OSI Model](https://www.practicalnetworking.net/series/packet-traveling/osi-model/) →
  [Through a Switch](https://www.practicalnetworking.net/series/packet-traveling/host-to-host-through-a-switch/) →
  [Through a Router](https://www.practicalnetworking.net/series/packet-traveling/host-to-host-through-a-router/) →
  [Host → Switch → Router → Switch → Host](https://www.practicalnetworking.net/series/packet-traveling/host-switch-router-switch-host/)
- Optional, if you like hardware: Ben Eater, *Networking tutorial* (bits on a wire up to IP): [playlist](https://www.youtube.com/playlist?list=PLowKtXNTBypH19whXTVoG3oKSuOcw_XeW)

**Do**
```bash
ip neigh                                   # your ARP cache
sudo tcpdump -i any -nn arp or icmp        # terminal 1
ping -c 1 <your gateway IP>                # terminal 2: watch ARP, then ICMP
```
Install [Wireshark](https://www.wireshark.org/), capture while opening a website, and click through *Ethernet → IP → TCP → HTTP*.

**Checkpoint:** draw the path of a packet from your PC to `8.8.8.8` and say, at each hop, which MAC and which IP are in the frame.

**Deeper:** [README §1–4](README.md#1-the-big-picture-osi-and-tcpip), [§10 full journey](README.md#10-full-journey-a-opens-a-web-page-on-c)

---

## Stage 5 — Routing, NAT and the Internet

**Goal:** build routing tables that work in both directions.

**Learn**
1. Routing table: destination network + gateway. `default` = `0.0.0.0/0`.
2. A router checks its own networks first, then its routes.
3. Static routes, default route, return path.
4. Router interfaces never overlap.
5. NAT: why private IPs still reach the Internet in real life (and why NetPractice has no NAT).
6. TTL and routing loops.

**Watch**
- Jeremy's IT Lab, *Free CCNA*: the days on routing fundamentals, static routing and NAT: [playlist](https://www.youtube.com/playlist?list=PLxbwE86jKRgMpuZuLBivzlM8s2Dk5lXBQ)

**Do**
- `traceroute 8.8.8.8`: see every router on the way.
- `curl ifconfig.me` vs `ip addr`: see NAT in action.

**Checkpoint:** ✔ **NetPractice levels 5–10** solved, then **evaluation mode** passed 3 times in a row.

**Deeper:** [README §6–7](README.md#6-layer-3-routing), [method](README.md#12-method-for-any-level), [mistakes](README.md#14-mistake-cheat-sheet)

---

# Applied

## Stage 6 — TCP, UDP and application protocols

**Goal:** understand what happens above IP when you open a website or chat.

**Learn**
1. Ports and sockets: the 4-tuple (src IP, src port, dst IP, dst port).
2. TCP: handshake, sequence numbers, ACKs, FIN/RST, stream (no message boundaries).
3. UDP: datagrams, no connection, fast.
4. DNS (names → IPs), DHCP (automatic IP config).
5. HTTP: request, response, headers, status codes.
6. TLS: what HTTPS adds.

**Watch / Read**
- Jeremy's IT Lab, *Free CCNA*: the days on TCP & UDP, DNS, DHCP: [playlist](https://www.youtube.com/playlist?list=PLxbwE86jKRgMpuZuLBivzlM8s2Dk5lXBQ)
- [MDN: HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)

**Do**
```bash
dig example.com                      # DNS over UDP
curl -v http://example.com/          # see the HTTP request and response
nc example.com 80                    # type: GET / HTTP/1.1, Host: example.com, empty line
```
In Wireshark, filter `tcp.flags.syn == 1` and find a 3-way handshake.

**Checkpoint:** explain why TCP can return `"hel"` + `"loworld"` for two sends `"hello"`, `"world"`, and what you must do about it.

**Deeper:** [README §8–9](README.md#8-layer-4-ports-tcp-and-udp)

---

## Stage 7 — Network programming (webserv / ft_irc)

**Goal:** write a TCP server in C/C++ that handles many clients at once.

**Learn**
1. `socket` → `bind` → `listen` → `accept` → `recv`/`send` → `close`.
2. Non-blocking sockets and `poll()` / `select()` / `epoll()` / `kqueue()`.
3. A buffer per client; framing messages (`\r\n`, `\r\n\r\n`, `Content-Length`).
4. HTTP/1.1 (webserv) or the IRC protocol (ft_irc).

**Read**
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/): read it fully, it's short and fun.
- `man 2 socket`, `man 2 poll`, `man 7 tcp`.
- webserv: [RFC 9112](https://www.rfc-editor.org/rfc/rfc9112), [NGINX docs](https://nginx.org/en/docs/).
- ft_irc: [Modern IRC docs](https://modern.ircdocs.horse/), [RFC 2812](https://www.rfc-editor.org/rfc/rfc2812).

**Do**
1. An echo server: send back whatever a client sends.
2. Make it handle 10 `nc` clients at once with `poll()`.
3. A mini chat: one client's message goes to all the others. (That's the heart of ft_irc.)

**Checkpoint:** ✔ webserv or ft_irc passes evaluation.

**Deeper:** [README §16–17](README.md#16-from-netpractice-to-webserv-and-ft_irc)

---

# Advanced

## Stage 8 — Switching deeper: VLANs, trunks, STP

**Goal:** design a real company LAN.

**Learn**
1. **VLAN**: split one switch into several virtual switches (staff, guests, servers). Different VLANs = different subnets, and a router or firewall decides what can cross.
2. **Trunk (802.1Q)**: one cable carrying many VLANs, tagged.
3. **Inter-VLAN routing**: "router on a stick" or a layer 3 switch.
4. **STP (Spanning Tree)**: stops loops when switches have redundant links (a switching loop can melt a network in seconds).
5. **EtherChannel**: bundle several cables into one fast link.

**Watch:** Jeremy's IT Lab, *Free CCNA*: the days on VLANs, DTP/VTP, STP, EtherChannel: [playlist](https://www.youtube.com/playlist?list=PLxbwE86jKRgMpuZuLBivzlM8s2Dk5lXBQ)

**Do:** [Cisco Packet Tracer](https://www.netacad.com/) (free): build 2 VLANs on one switch, then make them talk through a router.

**Checkpoint:** explain why two PCs on the same switch but different VLANs can't ping each other without a router.

---

## Stage 9 — Dynamic routing: RIP, OSPF, BGP

**Goal:** understand how routers learn routes **by themselves**, inside a company and across the whole Internet.

In NetPractice you typed every route by hand: **static routing**. With 5 routers that's fine; with 5,000 it's impossible. Routers therefore talk to each other with **routing protocols** and build their tables automatically.

### Two families

| | IGP (Interior Gateway Protocol) | EGP (Exterior Gateway Protocol) |
| --- | --- | --- |
| Where | **inside** one organisation | **between** organisations |
| Examples | RIP, OSPF, EIGRP, IS-IS | **BGP** (the only one used today) |
| Goal | fastest path | path allowed by business **policy** |

### RIP and OSPF (inside a company)

- **RIP**: "distance vector". Each router tells its neighbours "I can reach network X in N hops". Simple, slow, max 15 hops. Mostly historical, good for learning.
- **OSPF**: "link state". Every router learns the **full map** of the network, then computes the shortest path itself (Dijkstra's algorithm). Fast, scales well, splits big networks into **areas**. Used everywhere in companies.

### BGP: the protocol that holds the Internet together

**BGP (Border Gateway Protocol)** is how the networks that make up the Internet tell each other *"these IP addresses are reachable through me."*

**The key idea: Autonomous Systems (AS).** The Internet isn't one network; it's about 100,000 independent networks: ISPs, Google, Cloudflare, universities, 42's provider… Each one is an **Autonomous System** with a number (**ASN**), e.g. AS15169 = Google, AS13335 = Cloudflare.

**How it works, step by step**

```
            AS 100 (your ISP)  ─────  AS 200 (big transit ISP)  ─────  AS 300 (Google)
                                                                         owns 8.8.8.0/24
```

1. **Google (AS 300) announces:** "I own `8.8.8.0/24`, send traffic for it to me."
2. **AS 200 hears it** and tells its neighbours: "I can reach `8.8.8.0/24`, path = `200 300`."
3. **Your ISP (AS 100) hears it** and writes in its routing table: "`8.8.8.0/24` → via AS 200, path `200 300`."
4. You `ping 8.8.8.8` → your ISP forwards toward AS 200 → AS 200 forwards toward AS 300 → Google.

It's the same "destination network → next hop" table as NetPractice, but **learned automatically** from neighbours instead of typed by hand.

**The link with NetPractice:** remember the Internet's route in levels 6–10, `164.84.59.0/26 → 163.x.250.12`? In real life you never type that. Your ISP **announces your block with BGP** and the whole Internet learns where to send your replies. That fixed Internet route in NetPractice *is* what BGP does.

**How BGP chooses a path:** not just "shortest". It looks at:
- **AS path length**: fewer networks to cross is better.
- **Local preference**: "prefer the cheaper provider".
- **Business relationships**: customers pay providers; peers exchange traffic for free. Networks prefer routes that earn or save money.

**eBGP vs iBGP**
- **eBGP**: between two different ASes (at the border).
- **iBGP**: inside one AS, to share what the border routers learned.

**Why BGP is famous (and scary):** BGP trusts what neighbours announce. If someone announces addresses they don't own (a **BGP hijack**) or a network withdraws its own routes by mistake, traffic goes to the wrong place or disappears.
- **2008:** a Pakistani ISP announced YouTube's addresses to block it locally; the announcement leaked worldwide and YouTube went down globally for about two hours.
- **October 2021:** a configuration error made Facebook withdraw its BGP routes. Facebook, Instagram and WhatsApp vanished from the Internet for about six hours.
- The fix in progress: **RPKI**, cryptographically signing who owns which IP block.

**Explore real BGP**
- [bgp.he.net](https://bgp.he.net/): type `8.8.8.8` or `AS15169`. You'll see Google's ASN, the IP blocks it announces, and its neighbours.
- [Cloudflare Learning: What is BGP?](https://www.cloudflare.com/learning/security/glossary/what-is-bgp/)

**Watch:** Jeremy's IT Lab, *Free CCNA*: OSPF Part 1–3 (days 26–28) and the dynamic routing day: [playlist](https://www.youtube.com/playlist?list=PLxbwE86jKRgMpuZuLBivzlM8s2Dk5lXBQ)

**Do:** in Packet Tracer, replace the static routes of a NetPractice level (e.g. level 9) with OSPF, and watch the routing tables fill themselves (`show ip route`).

**Checkpoint:** explain in your own words what an AS is, what BGP announces, and why a single wrong announcement can take a website offline worldwide.

---

## Stage 10 — IPv6

**Goal:** be ready for the addresses that replace IPv4.

**Learn**
1. 128-bit addresses written in hex: `2001:db8:85a3::8a2e:370:7334`; `::` compresses zeros.
2. Prefixes work like CIDR: a home usually gets a `/56` or `/64`; one LAN = `/64`.
3. So many addresses that **no NAT is needed**: every device can be public (so firewalls matter more).
4. No ARP: **NDP** (Neighbor Discovery) does that job. No broadcast: multicast instead.
5. **SLAAC**: devices build their own address automatically.

**Do:** `ip -6 addr`, `ping -6 google.com`.

**Checkpoint:** shorten `2001:0db8:0000:0000:0000:0000:0000:0001` correctly (`2001:db8::1`).

---

## Stage 11 — Network security

**Goal:** know how networks are attacked and protected.

**Learn**
1. **Firewalls and ACLs**: allow or deny by IP, port, protocol, connection state. Try `ufw`, `nftables`.
2. **Segmentation**: subnets + VLANs + firewall between them. Changing your own IP doesn't bypass a firewall on the router.
3. **Layer 2 attacks**: ARP spoofing, MAC flooding, rogue DHCP. Defences: DAI, port security, DHCP snooping.
4. **TLS** and certificates: how HTTPS proves who the server is.
5. **VPN** (WireGuard, IPsec, OpenVPN): an encrypted tunnel across the Internet.
6. **IDS/IPS**: tools like Suricata and Snort that detect attacks in traffic.

**Do:** set up `ufw` on a VM: allow SSH and 8080, deny the rest, then test with `nc` from another machine.

**Checkpoint:** explain why "put the host in another subnet" is separation but not security.

---

## Stage 12 — Cloud and modern networks

**Goal:** recognise everything you learned inside today's infrastructure.

**Learn**
1. **Cloud VPCs** (AWS, GCP, Azure): you pick a CIDR like `10.0.0.0/16`, split it into subnets, and write route tables. **NetPractice skills, word for word.**
2. **Load balancers** and **reverse proxies** (NGINX, HAProxy): spread traffic across many servers.
3. **CDNs** (Cloudflare, Akamai): copies of content close to users, using BGP anycast.
4. **Docker / Kubernetes networking**: virtual bridges, NAT, overlay networks. `docker network inspect bridge` shows a subnet and a gateway.
5. **SDN** (Software-Defined Networking): networks configured by code instead of by hand.
6. **Linux network namespaces**: build a whole NetPractice topology on one Linux machine with `ip netns`.

**Do:** with `ip netns`, create two namespaces, connect them with a `veth` pair, give them IPs in the same /30 and ping between them. You've built NetPractice level 2 on your own kernel.

---

## Certifications (optional, if you want a career in networking)

| Level | Certification | Covers |
| --- | --- | --- |
| Beginner | CompTIA Network+ | stages 1–6, 10–11 basics |
| Associate | **Cisco CCNA 200-301** | stages 1–10 (Jeremy's IT Lab covers it for free) |
| Professional | Cisco CCNP Enterprise | deep OSPF, BGP, automation |
| Security | CompTIA Security+ | stage 11 |

---

## Quick glossary (words you'll keep hearing)

| Word | Meaning | Stage |
| --- | --- | --- |
| Host | any end device: PC, phone, server | 1 |
| Octet | 8 bits, one of the 4 numbers in an IPv4 address | 2 |
| CIDR | the `/n` notation for masks | 2–3 |
| Subnet | a piece of a bigger network | 3 |
| VLSM | subnets of different sizes from one block | 3 |
| Gateway | the router a host sends to for anything outside its network | 5 |
| MAC | layer 2 hardware address | 4 |
| ARP | finds the MAC for an IP on the local network | 4 |
| NAT | rewrites private IPs to a public one | 5 |
| TTL | hop counter that stops loops | 5 |
| Socket | a program's endpoint: IP + port | 6–7 |
| VLAN | a virtual switch inside a physical switch | 8 |
| STP | prevents switching loops | 8 |
| OSPF | routing protocol inside a company (link state) | 9 |
| AS / ASN | an independent network on the Internet and its number | 9 |
| BGP | the protocol ASes use to exchange routes: the Internet's routing | 9 |
| RPKI | signatures proving who owns an IP block | 9 |
| SLAAC | IPv6 automatic address setup | 10 |
| ACL | allow/deny rules on a router | 11 |
| VPC | your private network in the cloud | 12 |

---

## Your progress

Copy this into your notes and tick as you go:

- [ ] Stage 1: What is a network
- [ ] Stage 2: Binary and IP addresses
- [ ] Stage 3: Subnetting ✔ NetPractice 1–4
- [ ] Stage 4: How packets move
- [ ] Stage 5: Routing, NAT ✔ NetPractice 5–10 ✔ evaluation ×3
- [ ] Stage 6: TCP, UDP, applications
- [ ] Stage 7: Network programming ✔ webserv / ft_irc
- [ ] Stage 8: VLANs, STP
- [ ] Stage 9: OSPF, BGP
- [ ] Stage 10: IPv6
- [ ] Stage 11: Security
- [ ] Stage 12: Cloud and modern networks
