# 🐧 RHCSA Hands-On Self Challenge — Week 3 Part 1 Notes

> **Topic:** Networking Basics — Interfaces, IP Addressing, Routing, DNS
> **Environment:** CentOS 10 (VirtualBox VM)

---

## 📋 Table of Contents

- [Day 15 — Network Basics](#day-15--network-basics)
- [Quick Reference Card](#quick-reference-card)
- [Interview Questions](#interview-questions)

---

## Day 15 — Network Basics

### 🎯 Goal
> Understand network interfaces, IP addressing, subnets, gateways, routing, and DNS resolution — the foundation before configuring anything.

---

### 📖 Concepts

#### What is an IP Address?

A unique address that identifies a device on a network — like a street address so data can find its way to a device.

```
192.168.56.101
```

#### What is a Network Interface?

Before a device can have an IP address, it needs a **network interface** — the "door" through which data enters and exits.

> Think of a computer like a house. The network interface is the front door. The IP address is the address written on that door.

A computer can have multiple interfaces (multiple doors), each potentially leading to a different network.

```bash
ip a
```

Typical interfaces you'll see:

| Interface | Purpose |
|-----------|---------|
| `lo` | Loopback — talks to itself (127.0.0.1) |
| `enp0s3` | NAT network — internet access through host |
| `enp0s8` | Host-Only network — direct host ↔ VM communication |

---

#### What is Loopback (lo)?

```
inet 127.0.0.1/8
```

Every Linux system has this. It's how a computer talks to **itself** — used when a program needs to talk to another program on the SAME machine (e.g., a local database).

> Like calling your own phone number from your own phone — it never leaves the house, it just loops back to you.

---

#### Understanding Subnet Masks (/24 notation)

An IP address has 32 bits total. The `/24` (CIDR notation) tells you where the **network part** ends and the **device part** begins.

```
192      .168      .56       .101
11000000.10101000.00111000.01100101
└──────────24 bits──────────┘└─8 bits─┘
      Network part            Device part
```

> Think of it like a phone number: `192.168.56` is the area code (neighborhood), `.101` is the house number (specific device).

| Notation | Network Bits | Devices Available |
|----------|--------------|-------------------|
| /24 | 24 bits | 256 addresses (254 usable) |
| /16 | 16 bits | 65,536 addresses |
| /8 | 8 bits | 16 million addresses |

**Rule:** Devices in the SAME subnet (same `/24` network) can talk to each other directly. Devices in DIFFERENT subnets need a gateway.

---

#### What is a Gateway?

The gateway is the "exit road" out of your local network — the door to reach OTHER networks (like the internet or a different subnet).

> Think of your neighborhood having one main road connecting to the highway. Every car leaving to go elsewhere passes through that road. That's your gateway.

Check your routing table:

```bash
ip route
```

Example output:

```
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15 metric 100
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.101 metric 101
```

Reading it:

| Line | Meaning |
|------|---------|
| `default via 10.0.2.2 dev enp0s3` | Anything not on my local networks → send through gateway 10.0.2.2 |
| `10.0.2.0/24 dev enp0s3` | Devices starting 10.0.2.X are reachable directly via enp0s3 |
| `192.168.56.0/24 dev enp0s8` | Devices starting 192.168.56.X are reachable directly via enp0s8 |

**Rule of thumb:**
```
Same subnet      → reach directly, no gateway needed
Different subnet → need a gateway (exit road)
```

---

#### Real Example — Host and VM on Same Subnet

```
Windows/Linux host:  192.168.56.1     (on vboxnet0)
VM:                  192.168.56.101   (on enp0s8)

Both in 192.168.56.0/24 → SAME neighborhood → direct communication, no gateway needed!
```

This is exactly why `ssh pujan@192.168.56.101` works instantly — no gateway required.

Meanwhile, the host and VM each have their OWN separate default gateway for reaching the wider internet (different networks entirely):

```
Host's gateway: 192.168.1.254   (home WiFi router)
VM's gateway:   10.0.2.2        (VirtualBox virtual router)
```

---

#### Testing Connectivity with ping

```bash
ping -c 4 192.168.56.1
```

| Flag | Meaning |
|------|---------|
| `-c 4` | Count = send only 4 packets then stop (without this, ping runs forever) |

Reading the output:

```
4 packets transmitted, 4 received, 0% packet loss   ← connection is healthy
time=0.435 ms                                        ← latency (round trip time)
ttl=64                                                ← Time To Live (hop limit)
```

**Latency comparison:**

| Target | Latency | Why |
|--------|---------|-----|
| Same machine (host↔VM) | ~0.4ms | Virtually instant, same physical hardware |
| Internet (google.com) | ~80ms | Real round trip across the internet |

**What is TTL?**
Time To Live — decreases by 1 every time a packet passes through a router. Prevents packets from looping forever on a broken network. If it hits 0, the packet is discarded.

**Real world use of ping:**
- Is this server even reachable?
- Is the network down, or just one application?
- Is there packet loss (unstable connection)?

---

#### What is DNS?

Computers only understand IP addresses, not names like `google.com`. DNS (Domain Name System) translates human-friendly names into IP addresses.

> Think of DNS like a phone contacts list. You tap a friend's name, and your phone looks up the actual number. DNS does the same for websites.

```
You type:        ping google.com
DNS translates:  192.178.158.138 (or similar)
Then Linux connects to that IP
```

#### Where Linux Looks Up DNS

```bash
cat /etc/resolv.conf
```

```
# Generated by NetworkManager
search worldlink.com.np
nameserver 10.0.2.3
```

| Line | Meaning |
|------|---------|
| `nameserver 10.0.2.3` | The DNS server Linux asks to resolve names |
| `search worldlink.com.np` | Search domain — auto-appended to incomplete hostnames |

#### The Full DNS Flow

```
You type: ping google.com
      ↓
Linux checks /etc/resolv.conf → "ask 10.0.2.3 for help"
      ↓
DNS server responds: "google.com = 142.250.29.102"
      ↓
Linux now knows the IP and connects to it
```

---

#### Manually Querying DNS

```bash
# Simple, classic tool
nslookup google.com

# More detailed, preferred by sysadmins
dig google.com
```

**dig output breakdown:**

```
;; ANSWER SECTION:
google.com.  234  IN  A  142.250.29.139
```

| Part | Meaning |
|------|---------|
| `google.com.` | Domain being looked up |
| `234` | TTL — seconds this answer is cached before re-checking |
| `IN` | Internet class (always this) |
| `A` | Record type — "A" = IPv4 address |
| `142.250.29.139` | The actual answer |

**nslookup vs dig:**

| Tool | Best for |
|------|---------|
| `nslookup` | Quick, simple check |
| `dig` | Detailed technical output, preferred for real troubleshooting |

---

### 💻 Commands Summary

```bash
# View all network interfaces and their IPs
ip a

# View routing table (gateways and local networks)
ip route

# Test connectivity
ping -c 4 <ip-or-hostname>

# View DNS server configuration
cat /etc/resolv.conf

# Manually query DNS
nslookup google.com
dig google.com
```

---

## Quick Reference Card

```bash
# Interfaces and IPs
ip a
ip addr show

# Routing table / gateway
ip route
ip r

# Connectivity test
ping -c 4 <target>

# DNS config file
cat /etc/resolv.conf

# DNS lookup tools
nslookup <domain>
dig <domain>
```

---

## Interview Questions

### Basic Level

**Q: What is an IP address?**
> A unique address that identifies a device on a network, allowing data to be routed to the correct destination — similar to a street address for mail.

**Q: What is a network interface?**
> The "door" through which a device sends and receives network data. A device can have multiple interfaces (e.g., enp0s3, enp0s8), each potentially on a different network.

**Q: What is the loopback interface used for?**
> `lo` (127.0.0.1) allows a computer to communicate with itself — used when a local program needs to talk to another local program on the same machine.

**Q: What does /24 mean in an IP address like 192.168.56.101/24?**
> It's CIDR notation indicating the first 24 bits are the network portion and the remaining 8 bits identify individual devices, giving 256 possible addresses (254 usable) in that subnet.

**Q: What command shows all network interfaces and IP addresses?**
> `ip a` (or `ip addr show`)

---

### Intermediate Level

**Q: What is a gateway and when is it needed?**
> A gateway is the exit point to reach networks outside your local subnet. It's needed when communicating with a device on a DIFFERENT subnet. Devices on the SAME subnet communicate directly without a gateway.

**Q: What command shows the routing table, and what does the "default" route mean?**
> `ip route`. The "default" route defines where to send traffic when the destination doesn't match any other known local network — it goes through the default gateway.

**Q: If two devices are in the same /24 subnet, do they need a gateway to reach each other?**
> No. Devices in the same subnet can communicate directly since they're on the same local network segment.

**Q: What is DNS and why is it needed?**
> DNS (Domain Name System) translates human-readable domain names (like google.com) into IP addresses, since computers only communicate using IP addresses, not names.

**Q: Where does Linux look up which DNS server to use?**
> `/etc/resolv.conf` — this file lists the nameserver(s) Linux queries to resolve domain names.

**Q: What is the difference between nslookup and dig?**
> Both perform DNS lookups. `nslookup` is simpler and quicker for a basic check. `dig` provides more detailed technical output (TTL, query time, response flags) and is generally preferred for real troubleshooting.

---

### Scenario Based

**Q: A server can ping its default gateway but cannot reach the internet. What would you check?**
> Check `/etc/resolv.conf` for DNS configuration issues, verify the gateway itself has internet access, check firewall rules that might be blocking outbound traffic, and test with an IP directly (e.g., `ping 8.8.8.8`) to isolate whether it's a DNS problem or a routing/connectivity problem.

**Q: You can ping a server by IP address but not by hostname. What does this tell you?**
> This points to a DNS resolution problem, not a network connectivity problem. The network path works (proven by successful IP ping), but something is wrong with DNS — check `/etc/resolv.conf`, try `dig <hostname>` to see if DNS responds, or check if the DNS server itself is reachable.

**Q: Two VMs are on the same host but can't ping each other despite both having network adapters enabled. What might be wrong?**
> Likely they are on different networks/adapter types (e.g., one on NAT, one on Host-Only) rather than the same subnet. Check `ip a` on both to confirm they share a common subnet, and verify both interfaces are up with `ip a` showing `UP` status.

---

*Notes by Pujan | RHCSA Hands-On Self Challenge | Week 3 Part 1*
