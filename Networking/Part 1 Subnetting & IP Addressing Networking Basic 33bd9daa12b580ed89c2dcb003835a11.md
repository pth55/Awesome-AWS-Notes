# Part 1: Subnetting & IP Addressing : Networking Basics for AWS VPC

---

## Table of Contents

1. [Why Networking Matters for VPC](Part%201%20Subnetting%20&%20IP%20Addressing%20Networking%20Basic%2033bd9daa12b580ed89c2dcb003835a11.md)
2. [IP Addressing Fundamentals](Part%201%20Subnetting%20&%20IP%20Addressing%20Networking%20Basic%2033bd9daa12b580ed89c2dcb003835a11.md)
3. [Binary & The 32-Bit System](Part%201%20Subnetting%20&%20IP%20Addressing%20Networking%20Basic%2033bd9daa12b580ed89c2dcb003835a11.md)
4. [IPv4 Address Classes](Part%201%20Subnetting%20&%20IP%20Addressing%20Networking%20Basic%2033bd9daa12b580ed89c2dcb003835a11.md)
5. [Private vs Public IP Ranges](Part%201%20Subnetting%20&%20IP%20Addressing%20Networking%20Basic%2033bd9daa12b580ed89c2dcb003835a11.md)
6. [CIDR Notation Explained](Part%201%20Subnetting%20&%20IP%20Addressing%20Networking%20Basic%2033bd9daa12b580ed89c2dcb003835a11.md)
7. [Subnetting — Start from Zero](Part%201%20Subnetting%20&%20IP%20Addressing%20Networking%20Basic%2033bd9daa12b580ed89c2dcb003835a11.md)
8. [Dividing a Network — Practical Subnetting](Part%201%20Subnetting%20&%20IP%20Addressing%20Networking%20Basic%2033bd9daa12b580ed89c2dcb003835a11.md)
9. [Subnet Mask Deep Dive](Part%201%20Subnetting%20&%20IP%20Addressing%20Networking%20Basic%2033bd9daa12b580ed89c2dcb003835a11.md)
10. [Why AWS Reserves 5 IPs Per Subnet](Part%201%20Subnetting%20&%20IP%20Addressing%20Networking%20Basic%2033bd9daa12b580ed89c2dcb003835a11.md)
11. [Why AWS Allows Only /16 to /28](Part%201%20Subnetting%20&%20IP%20Addressing%20Networking%20Basic%2033bd9daa12b580ed89c2dcb003835a11.md)
12. [Key Networking Terms for VPC](Part%201%20Subnetting%20&%20IP%20Addressing%20Networking%20Basic%2033bd9daa12b580ed89c2dcb003835a11.md)
13. [Quick Reference Cheatsheet](Part%201%20Subnetting%20&%20IP%20Addressing%20Networking%20Basic%2033bd9daa12b580ed89c2dcb003835a11.md)

---

## 1. Why Networking Matters for VPC

A **Virtual Private Cloud (VPC)** is essentially your own private data center inside AWS — but it is entirely software-defined. Instead of physically cabling servers, you define networks, subnets, routing, and firewalls using IP addressing rules.

Everything in a VPC — EC2 instances, RDS databases, Lambda functions, load balancers — gets an IP address. If you don't understand how IP addressing and subnetting work, you will make planning mistakes that are painful to fix later because **you cannot resize a VPC or subnet after creation without recreating it**.

This is why subnetting knowledge is not optional — it is the foundation every VPC decision is built on.

---

## 2. IP Addressing Fundamentals

Every device on a network needs an address so other devices know how to reach it — exactly like a postal address for your house.

### Two types of IP addresses

**Public IP Addresses**

- Unique across the entire internet — no two devices on the internet share the same public IP
- Assigned by your Internet Service Provider (ISP)
- Used to communicate with anything outside your network (websites, APIs, other cloud services)
- Scarce resource — there are only ~4.3 billion IPv4 addresses in total, and we have essentially run out

**Private IP Addresses**

- Used only inside a local/internal network (home, office, or your VPC)
- Cannot be routed over the public internet — a router will never forward a private IP packet to the internet
- Can be reused by anyone — your home network and a company's office can both use `192.168.1.x` without conflict, because these packets never leave the private network
- This reuse is what has kept the internet functioning despite IPv4 exhaustion

---

## 3. Binary & The 32-Bit System

IPv4 addresses are 32 bits long. We write them in **dotted decimal** for human readability, but every router in the world works on the binary form underneath.

### From binary to decimal

An IP address has 4 groups (called octets), each containing 8 bits. Each bit position has a fixed value:

```
Bit position:   128   64   32   16    8    4    2    1
                  |    |    |    |    |    |    |    |
Binary:           1    1    0    0    0    0    0    0
Decimal:        128 + 64 +  0 +  0 +  0 +  0 +  0 +  0  =  192
```

Another example:

```
Bit position:   128   64   32   16    8    4    2    1
Binary:           1    0    1    0    1    0    0    0
Decimal:        128 +  0 + 32 +  0 +  8 +  0 +  0 +  0  =  168
```

So `192.168.1.0` in full binary is:

```
192      .    168      .      1      .      0
11000000 . 10101000 . 00000001 . 00000000
```

All 32 bits: `11000000 10101000 00000001 00000000`

You don't need to memorize binary conversion for daily VPC work, but you must understand **why the numbers are what they are** — especially 256, 128, 64, which come up constantly in subnetting.

---

## 4. IPv4 Address Classes

Before CIDR (modern notation), IP addresses were assigned in fixed classes. AWS and most modern systems use CIDR, but classes still matter because the **private IP ranges** used in VPCs come from these classes.

| Class | First Octet Range | Default Prefix | Max Networks | Max Hosts/Network |
|:------|:------------------|:---------------|:-------------|:------------------|
| A | 1 – 126 | /8 | 128 | 16,777,214 |
| B | 128 – 191 | /16 | 16,384 | 65,534 |
| C | 192 – 223 | /24 | 2,097,152 | 254 |
| D | 224 – 239 | N/A | (Multicast — no hosts) | |
| E | 240 – 255 | N/A | (Experimental — reserved) | |

**Class A** — Very few networks, enormous number of hosts. Think of large corporations or ISPs that were given a block like `10.x.x.x` and can use 16 million IPs within it.

**Class B** — Medium balance. A university or large company might have a Class B block.

**Class C** — Lots of small networks. A typical home router gives you a `192.168.x.x` block — a Class C private range — with up to 254 devices.

**Special:**

- `127.x.x.x` — Loopback. `127.0.0.1` always means "this machine itself." When you ping `127.0.0.1`, the packet never leaves your network card. Used for testing.
- `169.254.x.x` — Link-local. Automatically assigned when a device can't get a DHCP address. In AWS, `169.254.169.254` is the instance metadata endpoint.

---

## 5. Private vs Public IP Ranges

These three ranges are permanently reserved for private use by RFC 1918. They will never be assigned as public internet addresses:

| Range | CIDR | Class | Addresses Available |
|:------|:-----|:------|:--------------------|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | A | 16,777,216 |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | B | 1,048,576 |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | C | 65,536 |

**AWS VPC uses these ranges.** When you create a VPC you pick a CIDR block from one of these three private ranges. Most AWS tutorials and real-world VPCs use `10.0.0.0/16` because it gives you 65,536 addresses to carve up into subnets — plenty of room for any architecture.

**Why can't you use public IPs inside a VPC?**
You technically can assign any range, but using a public IP range internally would create a conflict — your instances inside the VPC wouldn't be able to reach the real servers on those public IPs because the traffic would stay inside your network instead of routing to the internet. Always use private ranges for VPC CIDR blocks.

---

## 6. CIDR Notation Explained

CIDR stands for **Classless Inter-Domain Routing**. It replaced the old class system and gives you precise control over how many addresses a network contains.

The format is: `IP address / prefix length`

```
192.168.1.0 / 24
      │          └── prefix length: how many bits are the NETWORK portion
      └────────────── the network address (starting IP)
```

The prefix length (the number after `/`) tells you:

- How many bits from the left are **fixed** (the network ID)
- How many bits from the right are **free** (the host IDs — your actual devices)

```
Total bits in IPv4 = 32

Network bits = prefix length
Host bits    = 32 - prefix length
```

### Calculating available addresses

```
Host bits = 32 - prefix
Addresses  = 2 ^ host bits
```

Examples:

```
/32  →  32 - 32 = 0 host bits  →  2^0  = 1 address     (single IP, used for routes/firewall rules)
/30  →  32 - 30 = 2 host bits  →  2^2  = 4 addresses   (point-to-point link, 2 usable)
/28  →  32 - 28 = 4 host bits  →  2^4  = 16 addresses  (smallest AWS subnet)
/27  →  32 - 27 = 5 host bits  →  2^5  = 32 addresses
/26  →  32 - 26 = 6 host bits  →  2^6  = 64 addresses
/25  →  32 - 25 = 7 host bits  →  2^7  = 128 addresses
/24  →  32 - 24 = 8 host bits  →  2^8  = 256 addresses
/23  →  32 - 23 = 9 host bits  →  2^9  = 512 addresses
/22  →  32 - 22 = 10 host bits →  2^10 = 1,024 addresses
/20  →  32 - 20 = 12 host bits →  2^12 = 4,096 addresses
/16  →  32 - 16 = 16 host bits →  2^16 = 65,536 addresses (largest AWS VPC)
```

The key insight: **every time the prefix length increases by 1, you cut the number of addresses in half.** Going from /24 to /25 halves the addresses from 256 to 128. Going from /24 to /26 quarters it to 64.

---

## 7. Subnetting — Start from Zero

Let's use `192.168.1.0/24` as our working example throughout this section.

### Step 1: Understand what you have before any subnetting

`192.168.1.0/24` is one single network with no division yet.

```
32 - 24 = 8 host bits
2^8     = 256 total addresses
```

The address range is:

```
First address:  192.168.1.0    (network address — identifies the network itself)
Last address:   192.168.1.255  (broadcast address — sends to all devices on network)
Usable range:   192.168.1.1  to  192.168.1.254  =  254 usable IPs
```

Think of `192.168.1.0/24` as a building with 256 rooms. Room 0 is the building's official address (you can't live there). Room 255 is the intercom that broadcasts to everyone (you can't live there either). That leaves 254 rooms for actual residents.

At this point, all 256 addresses are in one flat network. Every device can talk to every other device. There is no separation or isolation — one big open floor.

---

## 8. Dividing a Network — Practical Subnetting

Resource: https://www.davidc.net/sites/default/subnets/subnets.html

Subnetting is the act of **dividing a larger network into smaller networks**. The reason you do this in AWS is:

- Security — keep databases separate from web servers
- Availability — spread resources across Availability Zones
- Routing control — some subnets have internet access (public), others don't (private)

### The core rule of subnetting

When you subnet, you **borrow bits from the host portion and add them to the network portion**. This creates multiple smaller networks.

```
Number of subnets created = 2 ^ (new prefix - original prefix)
Addresses per subnet      = 2 ^ (32 - new prefix)
```

---

### Case 1: I want 2 subnets — use /25 from /24

**Requirement:** Split `192.168.1.0/24` into 2 equal subnets.

```
New prefix  = /25
Subnets     = 2 ^ (25 - 24) = 2^1 = 2 subnets  ✓
IPs each    = 2 ^ (32 - 25) = 2^7 = 128 addresses per subnet
```

```
Subnet 1:   192.168.1.0   to  192.168.1.127    →  192.168.1.0/25
            Network: .0         Broadcast: .127
            Usable:  192.168.1.1  to  192.168.1.126   (126 usable)

Subnet 2:   192.168.1.128  to  192.168.1.255   →  192.168.1.128/25
            Network: .128        Broadcast: .255
            Usable:  192.168.1.129 to  192.168.1.254  (126 usable)
```

Visual:

```
192.168.1.0/24  (256 addresses)
├── 192.168.1.0/25    → .0   to .127   (128 addresses, 126 usable)
└── 192.168.1.128/25  → .128 to .255   (128 addresses, 126 usable)
```

---

### Case 2: I want 3 subnets — not possible cleanly, use /26 for 4 subnets

**Requirement:** Split `192.168.1.0/24` into 3 subnets.

The important rule: **subnets must always be powers of 2** — you can have 2, 4, 8, 16 subnets, never 3, 5, or 7. This is because you are borrowing whole bits, and each bit doubles the count.

```
2 subnets  →  borrow 1 bit  →  /25
3 subnets  →  NOT POSSIBLE (not a power of 2)
4 subnets  →  borrow 2 bits →  /26   ← pick this, use 3 and leave 1 unused
8 subnets  →  borrow 3 bits →  /27
```

So for 3 subnets, you plan for 4 by using **/26** and leave the 4th subnet unallocated (or reserve it for future use).

**Calculating /26 from /24:**

```
New prefix  = /26
Subnets     = 2 ^ (26 - 24) = 2^2 = 4 subnets
IPs each    = 2 ^ (32 - 26) = 2^6 = 64 addresses per subnet

Total check: 4 subnets × 64 addresses = 256 addresses ✓ (matches the original /24)
```

**Finding each subnet's range:**

The block size (how far apart each subnet starts) = addresses per subnet = 64.

So subnets start at: 0, 64, 128, 192.

```
Subnet 1:   192.168.1.0    to  192.168.1.63    →  192.168.1.0/26
            Network: .0         Broadcast: .63
            Usable:  192.168.1.1   to  192.168.1.62   (62 usable)

Subnet 2:   192.168.1.64   to  192.168.1.127   →  192.168.1.64/26
            Network: .64        Broadcast: .127
            Usable:  192.168.1.65  to  192.168.1.126  (62 usable)

Subnet 3:   192.168.1.128  to  192.168.1.191   →  192.168.1.128/26
            Network: .128       Broadcast: .191
            Usable:  192.168.1.129 to  192.168.1.190  (62 usable)

Subnet 4:   192.168.1.192  to  192.168.1.255   →  192.168.1.192/26
            Network: .192       Broadcast: .255
            Usable:  192.168.1.193 to  192.168.1.254  (62 usable)
```

Visual breakdown:

```
192.168.1.0/24   (256 addresses total)
│
├── 192.168.1.0/26    →  .0   to  .63    (64 addr, 62 usable) ← use as Subnet A
├── 192.168.1.64/26   →  .64  to  .127   (64 addr, 62 usable) ← use as Subnet B
├── 192.168.1.128/26  →  .128 to  .191   (64 addr, 62 usable) ← use as Subnet C
└── 192.168.1.192/26  →  .192 to  .255   (64 addr, 62 usable) ← reserve for future
```

You use 3 and reserve 1. Nothing is wasted — the 4th block sits there available if you need it later.

---

### Case 3: I want 8 subnets — use /27 from /24

```
New prefix  = /27
Subnets     = 2 ^ (27 - 24) = 2^3 = 8 subnets
IPs each    = 2 ^ (32 - 27) = 2^5 = 32 addresses per subnet
Block size  = 32

Subnet starts: 0, 32, 64, 96, 128, 160, 192, 224
```

```
Subnet 1:  192.168.1.0/27    →  .0   to  .31
Subnet 2:  192.168.1.32/27   →  .32  to  .63
Subnet 3:  192.168.1.64/27   →  .64  to  .95
Subnet 4:  192.168.1.96/27   →  .96  to  .127
Subnet 5:  192.168.1.128/27  →  .128 to  .159
Subnet 6:  192.168.1.160/27  →  .160 to  .191
Subnet 7:  192.168.1.192/27  →  .192 to  .223
Subnet 8:  192.168.1.224/27  →  .224 to  .255
```

Each subnet has 32 addresses, with 30 usable (first and last reserved for network and broadcast).

---

### The subnetting formula — your mental model

```
Start with your base network, e.g. 192.168.1.0/24

Step 1: Decide how many subnets you need
        Round UP to nearest power of 2

Step 2: Find the new prefix
        new prefix = original prefix + log2(number of subnets)
        (or: keep adding 1 to prefix until 2^(new-original) >= subnets needed)

Step 3: Calculate addresses per subnet
        addresses = 2 ^ (32 - new prefix)

Step 4: Block size = addresses per subnet
        Subnets start at: 0, block, 2×block, 3×block ...

Step 5: Each subnet range = start to (start + block - 1)
        Network = first address, Broadcast = last address
        Usable  = everything in between
```

---

## 9. Subnet Mask Deep Dive

The subnet mask is an alternative way to express the prefix length. Instead of `/24`, you write `255.255.255.0`. They mean exactly the same thing.

### How subnet masks work

The mask is also 32 bits. Every `1` bit in the mask = network portion. Every `0` bit = host portion.

```
/24 in binary:   11111111 . 11111111 . 11111111 . 00000000
/24 in decimal:     255   .    255   .    255   .     0
```

```
/25 in binary:   11111111 . 11111111 . 11111111 . 10000000
/25 in decimal:     255   .    255   .    255   .   128
```

```
/26 in binary:   11111111 . 11111111 . 11111111 . 11000000
/26 in decimal:     255   .    255   .    255   .   192
```

```
/27 in binary:   11111111 . 11111111 . 11111111 . 11100000
/27 in decimal:     255   .    255   .    255   .   224
```

```
/28 in binary:   11111111 . 11111111 . 11111111 . 11110000
/28 in decimal:     255   .    255   .    255   .   240
```

### Prefix to subnet mask conversion table (last octet only for /24 and smaller)

| Prefix | Last Octet Binary | Decimal | Block Size | Subnets from /24 | IPs/Subnet |
|:-------|:------------------|:--------|:-----------|:-----------------|:-----------|
| /24 | 00000000 | 0 | 256 | 1 | 256 |
| /25 | 10000000 | 128 | 128 | 2 | 128 |
| /26 | 11000000 | 192 | 64 | 4 | 64 |
| /27 | 11100000 | 224 | 32 | 8 | 32 |
| /28 | 11110000 | 240 | 16 | 16 | 16 |
| /29 | 11111000 | 248 | 8 | 32 | 8 |
| /30 | 11111100 | 252 | 4 | 64 | 4 |
| /32 | 11111111 | 255 | 1 | 256 | 1 |

---

## 10. Why AWS Reserves 5 IPs Per Subnet

When you create any subnet in AWS, 5 IP addresses are automatically taken out of your usable pool. This is not optional — it happens for every subnet regardless of size.

Using `192.168.1.0/24` as the example (256 total, should give 254 usable, but AWS gives you 251):

| Address | Reserved For | Reason |
|:--------|:-------------|:-------|
| 192.168.1.0 | Network address | Standard — identifies the network itself. No device can ever use this address. This exists in all networks, not just AWS. |
| 192.168.1.1 | VPC Router | AWS places a virtual router at the first usable IP of every subnet. This is the default gateway your instances use to send traffic out of the subnet. |
| 192.168.1.2 | DNS Server | AWS runs a DNS resolver at the second usable IP (+2 from base). Instances use this to resolve domain names like ec2.amazonaws.com into IPs. Without DNS, AWS services like S3, RDS endpoints, etc. would be unreachable by name. |
| 192.168.1.3 | Future use | Reserved by AWS for future internal networking features. AWS does not use this today, but it's pre-allocated to avoid future migration headaches. |
| 192.168.1.255 | Broadcast address | Standard — the broadcast address sends a packet to every device on the subnet. AWS VPCs don't actually support broadcast (it's a virtual network), but the address is still reserved per networking convention. |

### Why does AWS even need these?

**The router (.1):** Every subnet needs a gateway — a doorway for traffic to leave the subnet and go elsewhere (to another subnet, to the internet, to an on-premises network). AWS puts this at `.1` by convention so instances always know where to send traffic destined for outside their subnet.

**The DNS resolver (.2):** AWS provides a managed DNS service called Route 53 Resolver that runs inside every VPC. Your EC2 instance needs to call something like `my-rds-instance.abc123.us-east-1.rds.amazonaws.com` and get an IP back. Without the DNS resolver at `.2`, this wouldn't work. Every time your instance makes an API call to AWS services, it is using this DNS resolver.

**Future use (.3):** Amazon has learned from experience — when they needed to add new infrastructure features over the years, modifying existing subnets was disruptive. By pre-reserving this address, they give themselves room to introduce new networking capabilities without requiring customers to recreate their subnets.

### Practical impact

| Subnet | Total IPs | AWS Reserves | Usable IPs |
|:-------|:----------|:-------------|:-----------|
| /28 | 16 | 5 | 11 |
| /27 | 32 | 5 | 27 |
| /26 | 64 | 5 | 59 |
| /25 | 128 | 5 | 123 |
| /24 | 256 | 5 | 251 |
| /23 | 512 | 5 | 507 |
| /22 | 1,024 | 5 | 1,019 |
| /16 | 65,536 | 5 | 65,531 |

For very small subnets this matters a lot. A `/28` gives you only 16 addresses and AWS takes 5, leaving you just 11. If you need a subnet for 12 EC2 instances, `/28` won't work — you'd need `/27` (27 usable).

---

## 11. Why AWS Allows Only /16 to /28

When creating a VPC or subnet, AWS enforces strict prefix length limits. You cannot go outside these boundaries:

| Component | Minimum | Maximum | Reason |
|:----------|:--------|:--------|:-------|
| VPC | /16 | /28 | See detailed explanation below |
| Subnet | /16 | /28 | Must be within the VPC CIDR range |

### Why /16 is the largest VPC allowed

`/16` = 65,536 IP addresses.

AWS does not allow `/8`, `/12`, or `/15` even though those are valid private ranges. The reasons:

**Routing table scale:** Every VPC maintains internal routing entries. A `/8` network (16 million IPs) would create routing table overhead that doesn't align with how AWS allocates and manages networking resources at scale. The `/16` limit keeps routing manageable while still providing more than enough IPs for any realistic workload.

**Resource planning discipline:** A /16 gives you 65,536 addresses. If you need more than that in a single VPC, you are almost certainly architecting something that should be multiple VPCs. AWS encourages using VPC Peering or Transit Gateway to connect multiple VPCs rather than building one massive flat network. Smaller, well-planned VPCs are easier to secure and manage.

**Overlap risk:** If AWS allowed `/8` VPCs, it would be very easy to overlap with corporate on-premises networks when setting up VPN or Direct Connect connections. `/16` forces reasonable scoping.

### Why /28 is the smallest subnet allowed

`/28` = 16 addresses, minus AWS's 5 reserved = **11 usable IPs**.

AWS does not allow `/29`, `/30`, `/31`, or `/32` subnets. The reasons:

**Minimum viable subnet:** With fewer than 11 usable IPs, a subnet becomes nearly unusable for any real purpose. Even a single EC2 instance with a load balancer and NAT Gateway would exhaust it. AWS sets `/28` as the floor to prevent customers from creating subnets too small to be useful.

**AWS-managed services need IPs too:** When you put an AWS managed service into a subnet — like an Application Load Balancer, RDS instance, NAT Gateway, or Lambda in VPC — AWS often needs to allocate additional IPs for internal use beyond your instance's IP. A subnet smaller than `/28` wouldn't reliably accommodate these needs.

**ENI (Elastic Network Interface) scaling:** EC2 instances can have multiple ENIs. Some instance types support 8+ ENIs. If your subnet is a `/30` with only 4 total addresses, you can't attach multiple network interfaces. `/28` provides enough headroom.

### The complete allowed range visualized

```
/16   →  65,536 total  →  65,531 usable  ← largest VPC
/17   →  32,768 total  →  32,763 usable
/18   →  16,384 total  →  16,379 usable
/19   →   8,192 total  →   8,187 usable
/20   →   4,096 total  →   4,091 usable
/21   →   2,048 total  →   2,043 usable
/22   →   1,024 total  →   1,019 usable
/23   →     512 total  →     507 usable
/24   →     256 total  →     251 usable  ← most common subnet size
/25   →     128 total  →     123 usable
/26   →      64 total  →      59 usable
/27   →      32 total  →      27 usable
/28   →      16 total  →      11 usable  ← smallest allowed
```

---

## 12. Key Networking Terms for VPC

These are terms you will encounter immediately when working with AWS VPC. Understanding them now prevents confusion later.

- **Network Address:** The very first IP in any subnet. It identifies the network itself and cannot be assigned to any device. For `192.168.1.0/24` this is `192.168.1.0`.
- **Broadcast Address:** The very last IP in any subnet. A packet sent to this address goes to every device on the subnet simultaneously. Cannot be assigned to a device. For `192.168.1.0/24` this is `192.168.1.255`. AWS VPCs don't use broadcast but still reserve this address.
- **Default Gateway:** The IP address of the router that devices use when they want to send traffic outside their own subnet. In AWS, this is always the `.1` address of each subnet (which AWS reserves). When you launch an EC2 instance and it wants to reach the internet or another subnet, it sends traffic to the default gateway first.
- **DHCP (Dynamic Host Configuration Protocol):** The protocol that automatically assigns IP addresses to devices when they join a network. In AWS, DHCP is managed automatically — when an EC2 instance launches, it receives its private IP via DHCP from the VPC. You do not configure DHCP manually in most cases.
- **DNS (Domain Name System):** Translates human-readable names like `google.com` into IP addresses. In AWS, the DNS resolver lives at the second IP of every subnet (e.g., `192.168.1.2`). AWS calls this the **Route 53 Resolver** or **Amazon DNS Server**. It is also always accessible at `169.254.169.253` from any instance.
- **Routing Table:** A set of rules that tells network traffic where to go. Every subnet in a VPC is associated with a routing table. A rule like `0.0.0.0/0 → igw-xxxxx` means "send all internet-bound traffic to the Internet Gateway." A rule like `10.0.2.0/24 → local` means "traffic for this subnet stays inside the VPC."
- **Public Subnet:** A subnet whose routing table has a route to an Internet Gateway (`0.0.0.0/0 → igw`). Resources here can reach the internet and be reached from the internet (if they also have a public IP).
- **Private Subnet:** A subnet with no route to an Internet Gateway. Resources here cannot be directly reached from the internet. They can still reach the internet outbound via a NAT Gateway placed in a public subnet.
- **CIDR Block:** The IP range assigned to your VPC or subnet, written in CIDR notation. Example: `10.0.0.0/16` for a VPC, `10.0.1.0/24` for a subnet within it.
- **Elastic IP (EIP):** A static public IPv4 address you own in AWS. Unlike the default public IPs that change every time an instance restarts, an Elastic IP stays the same. You associate it with an EC2 instance or NAT Gateway.
- **ENI (Elastic Network Interface):** A virtual network card attached to an EC2 instance. Each ENI gets a private IP from the subnet it is in. An instance can have multiple ENIs, allowing it to exist in multiple subnets simultaneously.
- **NAT Gateway:** Network Address Translation Gateway. Placed in a public subnet, it allows instances in private subnets to initiate outbound connections to the internet (for software updates, API calls, etc.) while blocking inbound connections from the internet. The NAT Gateway translates the private IP of your instance to its own public Elastic IP when sending traffic out.
- **Internet Gateway (IGW):** A horizontally scaled, redundant AWS-managed gateway attached to a VPC. It provides the route between your VPC and the public internet. Without an IGW, no subnet in your VPC can access the internet, regardless of routing table entries.
- **VPC Peering:** A networking connection between two VPCs that lets them communicate as if they were on the same network. The CIDR blocks of peered VPCs must not overlap — this is one reason careful IP planning matters upfront.
- **Availability Zone (AZ):** A physically separate data center within an AWS Region. Best practice is to create one subnet per AZ, spreading your resources for high availability. If one AZ fails, your resources in other AZs continue working.

---

## 13. Quick Reference Cheatsheet

### Subnetting from /24 at a glance

| Need X Subnets | Round to Power of 2 | Use Prefix | IPs/Subnet | Usable in AWS |
|:---------------|:--------------------|:-----------|:-----------|:--------------|
| 1 subnet | 1 (2^0) | /24 | 256 | 251 |
| 2 subnets | 2 (2^1) | /25 | 128 | 123 |
| 3 subnets | 4 (round up, 2^2) | /26 | 64 | 59 |
| 4 subnets | 4 (2^2) | /26 | 64 | 59 |
| 5–8 subnets | 8 (round up, 2^3) | /27 | 32 | 27 |
| 9–16 subnets | 16 (round up, 2^4) | /28 | 16 | 11 |

### Key formulas

```
Addresses in a block   =  2 ^ (32 - prefix)
Number of subnets      =  2 ^ (new prefix - old prefix)
Block size             =  addresses per subnet
Usable IPs             =  total addresses - 2 (NID, BID)
Usable IPs in AWS      =  total addresses - 5 (NID, VPC Router, DNS, Reserved, BID)
```

### Private IP ranges (use these for VPCs)

```
10.0.0.0/8         →  10.0.0.0    to  10.255.255.255
172.16.0.0/12      →  172.16.0.0  to  172.31.255.255
192.168.0.0/16     →  192.168.0.0 to  192.168.255.255
```

### AWS subnet reservation breakdown

```
.0    Network address (standard)
.1    VPC Router / Default Gateway (AWS)
.2    DNS Resolver (AWS)
.3    Future use (AWS)
.255  Broadcast address (standard, last IP in subnet)
```

### AWS VPC limits

```
Largest VPC:    /16  (65,536 addresses)
Smallest VPC:   /28  (16 addresses, 11 usable)
Smallest subnet: /28  (same limit)
Reserved per subnet: always 5 addresses
```

---