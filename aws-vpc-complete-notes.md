# AWS VPC — Complete Practical Notes (5-Hour Lecture Equivalent)

> **Goal:** After reading this and doing the labs, you should be able to design, build, secure, and troubleshoot a VPC — and answer SAA-C03 / DEA-C01 style VPC questions confidently.
>
> **How to use:** Read Part 1 (prerequisites) *without skipping*. Then read Part 2 (concepts) in order — each concept builds on the previous one. Then do the labs in Part 3 with your own AWS account. Finish with Parts 4–6 (patterns, exam sheet, practice questions).
>
> **Suggested time split:** Part 1 ≈ 40 min · Part 2 ≈ 100 min · Part 3 (labs) ≈ 120 min · Parts 4–6 ≈ 40 min.

---

## 📑 Table of Contents

- [Part 1 — Prerequisites (Networking from zero)](#part-1--prerequisites-networking-from-zero)
- [Part 2 — VPC Concepts in Depth](#part-2--vpc-concepts-in-depth)
- [Part 3 — Hands-On Labs (Live Practice)](#part-3--hands-on-labs-live-practice)
- [Part 4 — Real-World Architecture Patterns (incl. Data Engineering)](#part-4--real-world-architecture-patterns)
- [Part 5 — Exam Cheat Sheet](#part-5--exam-cheat-sheet)
- [Part 6 — Practice Questions with Explanations](#part-6--practice-questions-with-explanations)
- [Part 7 — Troubleshooting Playbook](#part-7--troubleshooting-playbook)
- [Part 8 — Final Revision Checklist](#part-8--final-revision-checklist)

---

# Part 1 — Prerequisites (Networking from zero)

You cannot understand VPC by memorizing. VPC is just **networking concepts + AWS names**. Learn the concepts first, and VPC will feel obvious.

## 1.1 What is a network?

A **network** is a group of computers that can talk to each other. To talk, every computer needs:

1. An **address** (who am I?) → **IP address**
2. A **path** (how do I reach others?) → **routing**
3. **Rules** (who is allowed to talk to me?) → **firewall**
4. A **name** (so humans don't type numbers) → **DNS**

A VPC gives you all four, inside AWS, completely under your control.

> 🏢 **Analogy for the whole note:** Think of a VPC as **your own private office campus** inside a giant city (AWS).
> - The **campus boundary** = VPC
> - **Buildings in different cities' zones** = Availability Zones
> - **Floors / wings** = Subnets
> - **Main gate to the outside world** = Internet Gateway
> - **Security guard at each room's door** = Security Group
> - **Security guard at each wing's entrance** = NACL
> - **Road signs that tell you where to go** = Route Tables
> - **A back door that lets employees go out but strangers can't come in** = NAT Gateway
> - **A private tunnel to another campus** = VPC Peering / Transit Gateway
> - **A private road to your head office** = VPN / Direct Connect

We will come back to this analogy again and again.

---

## 1.2 IP Address (IPv4)

An **IPv4 address** looks like `10.0.1.25`. It is **32 bits** divided into **4 octets** (8 bits each), written in decimal separated by dots.

```
10      .   0       .   1       .   25
00001010 . 00000000 . 00000001 . 00011001      ← actual binary (32 bits)
```

Each octet ranges from 0 to 255 (because 8 bits → 2⁸ = 256 values).

Total possible IPv4 addresses = 2³² ≈ **4.3 billion**. That's too few for the world → so we have **private IPs** (reused everywhere) and **public IPs** (globally unique).

### Public vs Private IP

| | Private IP | Public IP |
|---|---|---|
| Reachable from internet? | ❌ No | ✅ Yes |
| Unique worldwide? | ❌ No (your 10.0.1.5 and my 10.0.1.5 can both exist) | ✅ Yes |
| Cost | Free | AWS charges per public IPv4 address per hour (since 2024) |
| Used for | Communication inside VPC | Internet-facing resources |

### The three private ranges (RFC 1918) — MEMORIZE

| Range | CIDR | Total IPs | Typical use |
|---|---|---|---|
| 10.0.0.0 – 10.255.255.255 | `10.0.0.0/8` | ~16.7 million | Large enterprises, AWS VPCs |
| 172.16.0.0 – 172.31.255.255 | `172.16.0.0/12` | ~1 million | Medium networks (AWS **default VPC** uses 172.31.0.0/16) |
| 192.168.0.0 – 192.168.255.255 | `192.168.0.0/16` | 65,536 | Home routers, small networks |

> ⚠️ **Real-world tip:** Pick VPC CIDRs that **don't overlap** with your office network, other VPCs, or other accounts. Overlap = you can never connect them later (peering, VPN, TGW all break). Also avoid `172.17.0.0/16` — Docker uses it by default and causes confusing conflicts.

---

## 1.3 CIDR Notation (the most important prerequisite)

**CIDR** = Classless Inter-Domain Routing. It's how we write "a block of IP addresses".

Format: `IP-address/prefix-length` → example `10.0.0.0/16`

- The number after `/` = **how many bits are fixed (the network part)**.
- Remaining bits = **free to change (the host part)** = the addresses you get.

```
10.0.0.0/16
First 16 bits are FIXED  →  10.0 . xxxxxxxx . xxxxxxxx
Last 16 bits are FREE    →  2^16 = 65,536 addresses
Range: 10.0.0.0  to  10.0.255.255
```

**Formula:** Number of IPs = **2^(32 − prefix)**

### CIDR size table (learn this — used in exams and real work)

| CIDR | Free bits | Total IPs | AWS usable (minus 5) | Notes |
|---|---|---|---|---|
| /16 | 16 | 65,536 | 65,531 | **Largest VPC allowed** |
| /17 | 15 | 32,768 | 32,763 | |
| /18 | 14 | 16,384 | 16,379 | |
| /19 | 13 | 8,192 | 8,187 | |
| /20 | 12 | 4,096 | 4,091 | Default VPC subnet size |
| /21 | 11 | 2,048 | 2,043 | |
| /22 | 10 | 1,024 | 1,019 | |
| /23 | 9 | 512 | 507 | |
| /24 | 8 | 256 | **251** | Most common subnet size |
| /25 | 7 | 128 | 123 | |
| /26 | 6 | 64 | 59 | |
| /27 | 5 | 32 | 27 | |
| /28 | 4 | 16 | **11** | **Smallest VPC/subnet allowed** |
| /32 | 0 | 1 | — | A single IP (used in SG rules like `1.2.3.4/32`) |
| /0 | 32 | all | — | **Everything** (`0.0.0.0/0` = the whole internet) |

> 🧠 **Memory trick:** *Bigger prefix number = smaller network.* `/28` is tiny, `/16` is huge. Every time the prefix goes up by 1, the size halves.

### Worked example 1 — range of `10.0.1.0/24`

- First 24 bits fixed → `10.0.1.` is fixed.
- Last octet (8 bits) free → 0 to 255.
- Range: `10.0.1.0` → `10.0.1.255`, total 256 IPs.

### Worked example 2 — splitting a VPC into subnets

You own `10.0.0.0/16` (65,536 IPs). You want /24 subnets (256 IPs each).

```
65,536 / 256 = 256 possible subnets

10.0.0.0/24    10.0.1.0/24    10.0.2.0/24   ...   10.0.255.0/24
```

Our lab plan (we will build this later):

```
VPC:        10.0.0.0/16
public-a:   10.0.1.0/24     (AZ a)
public-b:   10.0.2.0/24     (AZ b)
private-a:  10.0.11.0/24    (AZ a)
private-b:  10.0.12.0/24    (AZ b)
```

### Worked example 3 — non-/24 subnets

Split `10.0.0.0/24` into four equal parts → need 4 pieces = 2 extra bits → /26:

```
10.0.0.0/26     → 10.0.0.0   – 10.0.0.63
10.0.0.64/26    → 10.0.0.64  – 10.0.0.127
10.0.0.128/26   → 10.0.0.128 – 10.0.0.191
10.0.0.192/26   → 10.0.0.192 – 10.0.0.255
```

### Rule: subnets inside a VPC must be inside the VPC CIDR and must NOT overlap each other.

---

## 1.4 Ports and Protocols

An IP address finds the **computer**. A **port** finds the **application** on that computer. (IP = building address, port = apartment number.)

- Port range: 0–65535.
- **TCP:** reliable, connection-based (3-way handshake). Used by web, SSH, databases.
- **UDP:** fast, no connection setup. Used by DNS (mostly), video, VoIP.
- **ICMP:** used by `ping` (no ports).

### Ports you MUST know

| Port | Protocol | Service |
|---|---|---|
| 22 | TCP | SSH (Linux login) |
| 3389 | TCP | RDP (Windows login) |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL / MariaDB / Aurora MySQL |
| 5432 | TCP | PostgreSQL / Aurora PostgreSQL / Redshift (default 5439) |
| 1521 | TCP | Oracle DB |
| 1433 | TCP | MS SQL Server |
| 5439 | TCP | Amazon Redshift |
| 6379 | TCP | Redis |
| 9092 / 9094 / 9096 / 9098 | TCP | Kafka / MSK (plaintext / TLS / SASL-SCRAM / IAM) |
| 2049 | TCP | NFS (EFS) |
| 53 | UDP/TCP | DNS |

### Ephemeral ports (important for NACLs!)

When your laptop connects to a web server on port 80, your laptop uses a **random high port** (e.g., 52341) as its *source* port. The server's reply goes **back to that random port**. These random ports are called **ephemeral ports** (roughly 1024–65535).

| OS | Ephemeral range |
|---|---|
| Linux | 32768 – 60999 |
| Windows (Server 2008+) | 49152 – 65535 |
| Common safe NACL rule (covers all) | **1024 – 65535** |

You will see why this matters in the NACL section — it's the #1 reason NACL questions confuse people.

---

## 1.5 Routers, Routing Tables, Gateways

- **Router:** a device that decides *where to forward* a packet.
- **Routing table:** the router's "road-sign list": *"for destination X, send to Y"*.
- **Default route** `0.0.0.0/0`: "for anything not matching a more specific rule, send here". Usually points to the internet gateway.
- **Longest prefix match:** if two routes match, the **more specific (longer prefix)** wins.

Example route table:

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |
| `10.1.0.0/16` | pcx-123 (peering) |
| `0.0.0.0/0` | igw-abc |

- Packet to `10.0.5.7` → matches `10.0.0.0/16` → stays local.
- Packet to `10.1.2.3` → matches `10.1.0.0/16` → goes via peering.
- Packet to `8.8.8.8` → only matches `0.0.0.0/0` → goes to internet gateway.
- If there is also `10.0.5.0/24 → X`, then `10.0.5.7` goes to X, because `/24` is more specific than `/16`.

---

## 1.6 NAT (Network Address Translation)

**Problem:** A server with only a private IP (10.0.11.5) wants to download a software update from the internet. Internet servers can't reply to a private IP.

**NAT solution:** a device with a public IP **replaces the source address** with its own public IP, sends the request, remembers the mapping, and when the reply returns, translates it back to the private server.

```
Private server 10.0.11.5 ──► [NAT device: public IP 13.x.x.x] ──► Internet
                         ◄──                                  ◄── (reply)
```

Key property: **outbound connections work; unsolicited inbound connections from the internet cannot reach the private server.** This is how we keep databases and app servers hidden but still able to download patches.

---

## 1.7 Firewalls — Stateful vs Stateless (critical concept)

A firewall allows or blocks traffic based on rules (source, destination, port, protocol).

### Stateful firewall
- **Remembers connections.**
- If it allowed the request in, it **automatically allows the reply out** (no separate rule needed).
- AWS example: **Security Group**.

### Stateless firewall
- **Remembers nothing.** Each packet is judged alone.
- You must write rules for **both directions** (request and reply).
- AWS example: **Network ACL (NACL)**.

> 🧠 **Analogy:**
> - *Stateful* = hotel reception: "You walked in as a guest, so you're allowed to walk out too."
> - *Stateless* = airport immigration: "Show your documents every time you enter AND every time you exit."

---

## 1.8 DNS (Domain Name System)

DNS converts names to IPs: `google.com → 142.250.x.x`. In a VPC, AWS provides a DNS resolver so `ip-10-0-1-25.ap-south-1.compute.internal` or an S3 endpoint name resolves automatically. Remember this when we cover **VPC endpoints** and **DNS attributes**.

---

## 1.9 DHCP (very short)

DHCP automatically gives a new machine its IP address, DNS server, and domain name. In AWS, when you launch an EC2, DHCP gives it a private IP from the subnet. AWS lets you customize this with a **DHCP option set** (custom DNS server, domain name, NTP).

---

## 1.10 AWS Global Infrastructure (prerequisite for VPC scope)

| Term | Meaning | VPC relevance |
|---|---|---|
| **Region** | A geographic area (e.g., `ap-south-1` = Mumbai) with multiple AZs | **A VPC lives in ONE region** |
| **Availability Zone (AZ)** | One or more isolated data centers within a Region (e.g., `ap-south-1a`) | **A subnet lives in ONE AZ** |
| **Edge location** | CDN / entry points (CloudFront, Route 53) | Not part of VPC |

### 🎯 Scope rule (appears in exams!)

```
Region   ──► contains ──►  VPC   (VPC spans all AZs of a region, but only ONE region)
AZ       ──► contains ──►  Subnet (a subnet can NEVER span multiple AZs)
Subnet   ──► contains ──►  EC2, RDS, Lambda ENIs, etc.
```

Why does this matter? **High availability** = put the same app in subnets in **at least 2 AZs**.

---

## 1.11 IAM and Shared Responsibility (30-second recap)

- **IAM** controls *who can call AWS APIs* (e.g., who can create a VPC).
- **Security Groups/NACLs** control *which network traffic is allowed*.
- They are different layers. IAM ≠ network firewall.
- AWS secures the physical network; **you** configure SGs, NACLs, routes, and encryption.

---

## 1.12 ✅ Prerequisite Self-Test

Answer without looking back:

1. How many IPs are in a `/24`? **(256)**
2. What does `0.0.0.0/0` mean? **(Every IPv4 address / the internet)**
3. Which firewall type remembers connections? **(Stateful)**
4. Can a subnet span two AZs? **(No)**
5. Which route wins: `10.0.0.0/16` or `10.0.5.0/24` for 10.0.5.9? **(/24 — longest prefix)**
6. Is `172.32.0.1` a private IP? **(No — private range ends at 172.31.255.255)**

If you got all 6, proceed.

---

# Part 2 — VPC Concepts in Depth

## 2.1 What is a VPC?

**Amazon Virtual Private Cloud (VPC)** is a **logically isolated virtual network** in AWS where you launch resources (EC2, RDS, Redshift, MSK, Lambda, ECS, EKS, etc.). You control:

- IP address range (CIDR)
- Subnets
- Route tables
- Gateways (internet, NAT, VPN)
- Firewalls (Security Groups, NACLs)
- DNS settings

> 🏢 Analogy: You rent a plot of land (VPC) in AWS city. Nobody else can enter unless you build gates and give permission.

### Key facts

| Fact | Value |
|---|---|
| Scope | **Regional** (spans all AZs in one region) |
| CIDR size | Between **/16 and /28** |
| Default quota | **5 VPCs per region** (soft limit — can be increased) |
| Subnets per VPC | 200 (default, adjustable) |
| Secondary CIDRs | You can **add** more CIDR blocks later (up to 5 by default). You **cannot resize or edit** the primary CIDR. |
| Tenancy | `default` (shared hardware) or `dedicated` (single-tenant hardware, expensive) |
| Cost | **VPC itself is free**. You pay for NAT Gateway, VPN, endpoints (interface), public IPv4, data transfer, etc. |

> ⚠️ **Planning matters:** You can add a secondary CIDR, but you can't shrink or change the original. Give your VPC enough room (`/16` is common for production).

### Default VPC vs Custom VPC

| | Default VPC | Custom VPC |
|---|---|---|
| Created by | AWS automatically in every region | You |
| CIDR | `172.31.0.0/16` | You choose |
| Subnets | One **public** `/20` subnet per AZ | You design |
| Internet Gateway | Already attached | You attach |
| Route table | Main RT already routes `0.0.0.0/0` → IGW | You configure |
| Auto-assign public IP | **Yes** | **No** (by default) |
| Use case | Quick tests | **Production** |

> 🎯 **Exam:** If you delete the default VPC, you can **recreate it** from the console/CLI (`aws ec2 create-default-vpc`).

---

## 2.2 Subnets

A **subnet** is a **range of IPs inside the VPC, tied to ONE Availability Zone**.

### Public subnet vs Private subnet — the REAL definition

> A subnet is **public** if its **route table has a route `0.0.0.0/0` → Internet Gateway**.
> A subnet is **private** if it has **no route to an Internet Gateway**.

There is **no checkbox** called "public". It is decided *only* by the route table. (Very common exam trap.)

For an instance in a public subnet to *actually* reach the internet, you need **all four**:

1. VPC has an **Internet Gateway** attached
2. Subnet's route table has **`0.0.0.0/0 → igw-xxx`**
3. Instance has a **public IPv4 (or Elastic IP)** (or IPv6 address)
4. **Security Group + NACL** allow the traffic

Miss any one → no internet. (This is the #1 troubleshooting checklist.)

### Reserved IPs in every subnet — 5 addresses

For subnet `10.0.1.0/24`:

| Address | Purpose |
|---|---|
| `10.0.1.0` | Network address |
| `10.0.1.1` | VPC router |
| `10.0.1.2` | Amazon-provided DNS (VPC base + 2) |
| `10.0.1.3` | Reserved for future use |
| `10.0.1.255` | Broadcast (not supported in VPC but reserved) |

So a `/24` gives **251 usable IPs**, not 256. A `/28` gives only **11**.

### Subnet attributes

- **Auto-assign public IPv4:** if ON, every instance launched here gets a public IP automatically.
- **Auto-assign IPv6.**
- A subnet can have exactly **one route table** associated at a time. A route table can be associated with **many subnets**.
- A subnet can have exactly **one NACL**. A NACL can be associated with many subnets.

### Design tip — tiers

A typical 3-tier layout (per AZ):

| Tier | Subnet type | Contains |
|---|---|---|
| Web / Load balancer | Public | ALB, NAT GW, Bastion |
| App | Private | EC2 / ECS / Lambda ENIs |
| Data | Private (no internet route at all) | RDS, Redshift, MSK, ElastiCache |

---

## 2.3 Route Tables

A **route table (RT)** is a set of rules (routes) telling the VPC router where to send traffic from a subnet.

### Important facts

1. Every VPC has a **main route table** (created automatically).
2. Any subnet not explicitly associated with a custom RT uses the **main route table** (implicit association).
3. Every route table has an undeletable **local route**: `<VPC CIDR> → local`. This lets all subnets in the VPC talk to each other **by default**.
4. **Longest prefix match** decides which route is used.
5. You **cannot override the local route**: traffic destined for an address inside the VPC CIDR always stays local, even if you add another route for that range.
6. **Best practice:** Don't put an IGW route in the *main* route table. Leave main RT private and create a **custom public RT** — so new subnets are private by default (secure by default).

### Example public route table

| Destination | Target |
|---|---|
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | igw-0abc123 |

### Example private route table (with NAT)

| Destination | Target |
|---|---|
| 10.0.0.0/16 | local |
| 0.0.0.0/0 | nat-0def456 |
| pl-xxxx (S3 prefix list) | vpce-0s3gw (gateway endpoint) |

### Route targets you will see

`local`, `igw-…`, `nat-…`, `vgw-…`, `tgw-…`, `pcx-…` (peering), `vpce-…` (gateway endpoint), `eni-…` / `i-…` (appliance), `eigw-…` (egress-only IGW), `cgw`/VPN via VGW.

### Route propagation
For VPN/Direct Connect via a **Virtual Private Gateway**, you can enable **route propagation** so BGP-learned on-prem routes automatically appear in the RT.

---

## 2.4 Internet Gateway (IGW)

- A **horizontally scaled, redundant, highly available** VPC component that connects the VPC to the internet.
- **No bandwidth limit, no cost** (you pay only for data transfer & public IPv4).
- **One IGW per VPC**, and an IGW can be attached to only one VPC at a time.
- It performs **1:1 NAT** between an instance's **private IP ↔ public IP**.

> 🔍 Notice: the EC2 OS **never sees** its public IP (`ifconfig` shows only the private IP). The IGW does the translation. (Exam favorite!)

### Public IP vs Elastic IP vs Auto-assigned

| Type | Behavior |
|---|---|
| **Auto-assigned public IPv4** | Taken from AWS pool. **Changes if you stop/start** the instance. Released when instance terminates. |
| **Elastic IP (EIP)** | Static public IPv4 you **own** until you release it. Can be remapped to another instance. Retained on stop/start. |

> 💰 Since February 2024, **all public IPv4 addresses are charged** (~$0.005/hour each) — including those attached to running instances. Idle EIPs cost the same. Prefer private architecture + load balancers; use IPv6 where possible.

---

## 2.5 Elastic Network Interface (ENI)

An **ENI** is a virtual network card. Every EC2 has at least one (primary, `eth0`). Many services (Lambda in VPC, RDS, NAT GW, interface endpoints, Glue, MSK) place ENIs in your subnets.

Attributes of an ENI: one primary private IPv4, one or more secondary IPs, one EIP per private IP, one public IPv4, MAC address, one or more **security groups**, **source/destination check** flag, IPv6 addresses.

- An ENI lives in **one subnet / one AZ** and cannot be moved to another AZ.
- You can detach a secondary ENI and attach it to another instance **in the same AZ** → classic **failover trick** (keep same private IP/MAC).

---

## 2.6 Security Groups (SG) — the instance-level firewall

### Properties (memorize every row)

| Property | Value |
|---|---|
| Level | **ENI / instance level** |
| State | **Stateful** (reply traffic automatically allowed) |
| Rules | **Allow rules only** — you **cannot** write a deny rule |
| Default inbound | **Deny all** |
| Default outbound | **Allow all** |
| Evaluation | **All rules evaluated together**; if any allows, traffic is allowed (no order) |
| Source/destination can be | CIDR, a prefix list, **another security group**, or itself |
| Scope | Specific to a **VPC** (can't be used across VPCs; can reference peered VPC's SG in same region) |
| Change effect | **Immediate** — no restart |
| Attach | Many SGs per ENI (default 5), many ENIs per SG |

### 🔑 The superpower — referencing another SG

Instead of IP addresses, say "allow port 3306 from members of `app-sg`":

| SG | Inbound rule |
|---|---|
| `alb-sg` | TCP 443 from `0.0.0.0/0` |
| `app-sg` | TCP 8080 from `alb-sg` |
| `db-sg` | TCP 3306 from `app-sg` |

Now it doesn't matter how many app servers exist or what their IPs are (auto scaling!). Only members of `app-sg` can reach the DB. This is the **#1 production security pattern**.

### Common mistakes
- Opening SSH `22` to `0.0.0.0/0` → use your IP `/32` or better, use **SSM Session Manager** (no inbound port needed).
- Thinking you can "block an IP" with an SG → ❌ impossible; use **NACL** (or WAF/Network Firewall).
- Forgetting that SG rules are attached to the ENI, so *the instance's SG* is what's checked, not the subnet's.

---

## 2.7 Network ACLs (NACL) — the subnet-level firewall

### Properties

| Property | Value |
|---|---|
| Level | **Subnet level** (applies to all instances in that subnet) |
| State | **Stateless** (must allow both request and reply) |
| Rules | **Allow AND Deny** |
| Evaluation | **In order of rule number** (lowest first, 1–32766); **first match wins**, then stops |
| Default NACL (created with VPC) | **Allows ALL** inbound & outbound |
| New custom NACL | **Denies ALL** inbound & outbound until you add rules |
| Last rule `*` | Implicit **deny all** (can't be deleted) |
| Association | One NACL per subnet; one NACL can cover many subnets |
| Applies to | Traffic **entering or leaving the subnet** (subnet boundary only) |

> ✅ Correct statement for exam: **NACLs filter traffic crossing subnet boundaries**; traffic between two instances in the *same* subnet is not evaluated by the NACL (only by SGs).

### Stateless in action — the ephemeral port trap

Allow users to hit a web server in a custom NACL:

**Inbound rules**

| Rule # | Type | Protocol | Port | Source | Allow/Deny |
|---|---|---|---|---|---|
| 100 | HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 110 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| * | All | All | All | 0.0.0.0/0 | DENY |

**Outbound rules** (the reply!)

| Rule # | Type | Protocol | Port | Dest | Allow/Deny |
|---|---|---|---|---|---|
| 100 | Custom TCP | TCP | **1024–65535** | 0.0.0.0/0 | ALLOW |
| * | All | All | All | 0.0.0.0/0 | DENY |

Without the **outbound ephemeral rule**, requests arrive but replies are dropped → "site hangs/timeouts". That's the classic NACL bug.

### Blocking a bad IP (only NACL can do this)

| Rule # | Source | Action |
|---|---|---|
| **90** | 203.0.113.50/32 | **DENY** all |
| 100 | 0.0.0.0/0 | ALLOW 443 |

Rule 90 is evaluated before 100, so that IP is blocked. **Use numbers with gaps (10, 20, 30… or 100, 200…)** so you can insert rules later.

### SG vs NACL — the table that appears in every exam

| | Security Group | NACL |
|---|---|---|
| Works at | Instance / ENI | Subnet |
| Stateful? | **Yes** | **No** |
| Allow / Deny | Allow only | Allow + Deny |
| Rule evaluation | All rules | In number order, first match |
| Default | Inbound deny, outbound allow | Default NACL allows all; custom denies all |
| Can block specific IP? | ❌ | ✅ |
| Applies to | Only instances it's attached to | All instances in the subnet |
| Refer to other SG | ✅ | ❌ (only CIDRs) |

### Order of checks for traffic **coming IN** to an instance in a public subnet

```
Internet → IGW → Route table (VPC router) → NACL (inbound) → Security Group (inbound) → Instance
Reply:    Instance → SG (auto-allowed, stateful) → NACL (outbound) → Route table → IGW → Internet
```

> 🧠 *Two layers = defense in depth.* Both must allow for traffic to pass.

---

## 2.8 NAT Gateway (and NAT Instance)

**Purpose:** let instances in **private subnets** reach the internet (patches, APIs, pip/npm) **without** being reachable from the internet.

### NAT Gateway — managed (use this)

| Property | Value |
|---|---|
| Lives in | A **public subnet** (it needs a route to IGW) |
| Needs | An **Elastic IP** (for public NAT) |
| Scope | **One AZ** (zonal) — **not** automatically multi-AZ |
| HA | AWS makes it redundant **within the AZ**. For AZ failure, create **one NAT GW per AZ** and route each private subnet to its own AZ's NAT GW |
| Bandwidth | Starts at 5 Gbps, scales automatically up to 100 Gbps |
| Security Group | **Cannot** attach SG to NAT GW (use NACL on its subnet) |
| Protocols | TCP, UDP, ICMP |
| Cost | **Hourly charge + per GB processed** (this is why NAT is the "surprise bill" service) |
| Private NAT GW | Variant without EIP/IGW — used to connect to other VPCs/on-prem with overlapping/private ranges |

Route in private subnet's RT: `0.0.0.0/0 → nat-xxxx`
Route in public subnet's RT: `0.0.0.0/0 → igw-xxxx` (so the NAT GW itself can reach internet)

```
Private EC2 (10.0.11.5) ──► NAT GW (in public subnet, EIP 13.x.x.x) ──► IGW ──► Internet
```

> 🆕 AWS has also introduced a *regional* NAT Gateway option that spans AZs automatically. Check the current console/docs for availability — for exams, the classic "one zonal NAT GW per AZ" design is still the expected answer.

### NAT Instance — legacy (self-managed EC2)

- You launch an EC2 from a NAT AMI/config, in a public subnet.
- **Must disable Source/Destination Check** (otherwise the instance drops traffic not destined to itself).
- Pros: cheaper for tiny workloads, can use SGs, can double as bastion, port forwarding.
- Cons: you handle HA, patching, scaling, bandwidth limited by instance type.

### NAT GW vs NAT Instance (exam table)

| | NAT Gateway | NAT Instance |
|---|---|---|
| Managed by | AWS | You |
| Availability | HA within AZ | Manual (scripts/ASG) |
| Bandwidth | Up to 100 Gbps | Depends on instance type |
| Security Groups | ❌ | ✅ |
| Source/Dest check | N/A | **Disable required** |
| Bastion use | ❌ | ✅ |
| Port forwarding | ❌ | ✅ |

### 💡 Cost-saving trick (real-world)
If private instances only talk to **S3 / DynamoDB**, use **Gateway Endpoints (free)** instead of pushing that traffic through NAT (which charges per GB). This single change often cuts NAT bills massively for data pipelines.

---

## 2.9 Egress-Only Internet Gateway & IPv6

- IPv6 addresses in AWS are **all public & globally unique** (no private IPv6 / no NAT needed).
- A VPC gets a **/56** IPv6 block from Amazon; each subnet gets a **/64**.
- An **IGW** allows two-way IPv6.
- An **Egress-Only Internet Gateway (EIGW)** = *IPv6-only* "NAT-like" gateway: **outbound allowed, inbound initiated from internet blocked**. Route: `::/0 → eigw-xxxx`.
- You can run **dual-stack** (IPv4 + IPv6) or IPv6-only subnets.

> 🎯 Exam: "Private subnet instances using IPv6 need outbound internet only" → **Egress-only Internet Gateway**. "IPv4" → NAT Gateway.

---

## 2.10 DNS in a VPC & DHCP Option Sets

Two VPC attributes control DNS:

| Attribute | Meaning | Default |
|---|---|---|
| `enableDnsSupport` | Enables the Amazon DNS resolver (VPC base IP + 2, also `169.254.169.253`) | **true** |
| `enableDnsHostnames` | Gives instances with public IPs a **public DNS hostname** | **true in default VPC; false in custom VPCs you create (unless enabled)** |

Both must be **true** for Interface Endpoint **private DNS** and for **Route 53 private hosted zones** to work.

### DHCP option set
Defines DNS servers, domain name, NTP, NetBIOS for the VPC. **You cannot edit an existing option set** — create a new one and associate it with the VPC. One VPC ↔ one option set at a time.

### Route 53 Resolver
- Default resolver answers queries from inside the VPC.
- **Inbound endpoint:** on-prem → resolve AWS private names.
- **Outbound endpoint + forwarding rules:** AWS → resolve on-prem names.

---

## 2.11 VPC Endpoints (Private access to AWS services) — heavy exam & data-engineering topic

**Problem:** A private EC2/Glue job needs to reach **S3**. Without endpoints the traffic goes: private subnet → NAT → IGW → public internet path → S3. That costs money and leaves your private network (though still within AWS backbone, it uses public endpoints).

**Solution:** **VPC Endpoints** give private connectivity to AWS services **without IGW, NAT, VPN, or public IP**.

### Type 1 — Gateway Endpoint

| Property | Value |
|---|---|
| Supported services | **Only S3 and DynamoDB** |
| Cost | **Free** |
| How it works | Adds a **route** (prefix-list → vpce) to your chosen route tables |
| Has ENI/IP/SG? | ❌ No |
| Reachable from on-prem / peered VPC / TGW? | ❌ **No** (only from within that VPC) |
| Policy | Endpoint policy (resource-based) can restrict which buckets/actions |
| Region | Same-region service only |

### Type 2 — Interface Endpoint (powered by AWS PrivateLink)

| Property | Value |
|---|---|
| Supported services | **Most AWS services** (SQS, SNS, Kinesis, Secrets Manager, SSM, ECR, STS, CloudWatch, Glue, KMS, API Gateway, Athena, etc.) + your own services + 3rd-party SaaS |
| How it works | Creates an **ENI with private IP** in your subnet(s) |
| Cost | **Per hour per AZ + per GB** |
| Security Group | ✅ Attach SG (control who can use it) |
| Private DNS | If enabled, the normal service name (e.g., `sqs.ap-south-1.amazonaws.com`) resolves to the private IP automatically |
| Reachable from on-prem/peered/TGW | ✅ **Yes** (over VPN/DX) |
| HA | Create in **multiple AZs** |

### Type 3 — Gateway Load Balancer Endpoint
Used to route traffic through **third-party firewall/inspection appliances** (GWLB). (Just recognize it.)

### Gateway vs Interface — the decision

```
Need S3 or DynamoDB from inside the VPC only, cheapest?        → Gateway Endpoint
Need S3 access from on-premises over VPN/Direct Connect?        → Interface Endpoint for S3
Any other AWS service privately (SQS, KMS, Glue, SSM, ...)?     → Interface Endpoint
Expose MY service privately to other VPCs/accounts?             → PrivateLink (NLB + endpoint service)
```

### Endpoint policy example (allow only one bucket)

```json
{
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-company-data-lake/*"
  }]
}
```

Combine with an **S3 bucket policy** using `aws:SourceVpce` or `aws:SourceVpc` so the bucket can **only** be accessed through your endpoint — a very common compliance (BFSI) control.

---

## 2.12 VPC Peering

**Peering** = a private, 1:1 network connection between two VPCs (same or different account, same or different region) using AWS backbone.

### Rules (exam gold)

1. **No overlapping CIDRs** between the two VPCs.
2. **Non-transitive:** if A↔B and B↔C, **A cannot reach C** through B. You need A↔C directly.
3. **No edge-to-edge routing:** A can't use B's IGW, NAT GW, VPN, Direct Connect, or gateway endpoints.
4. You must **update route tables on both sides** (`10.1.0.0/16 → pcx-xxx` in VPC A and `10.0.0.0/16 → pcx-xxx` in VPC B).
5. You must **update SGs/NACLs** to allow traffic. Same-region peered SG referencing is supported.
6. Process: **Requester sends request → Accepter accepts** (cross-account needs accepter's account ID).
7. Cross-region peering is supported (traffic encrypted, never on public internet).
8. No bandwidth bottleneck / single point of failure — it's not a gateway or appliance.

```
VPC-A (10.0.0.0/16) ◄──pcx──► VPC-B (10.1.0.0/16) ◄──pcx──► VPC-C (10.2.0.0/16)
        A ✗───────────── cannot reach C via B ─────────────✗ C
```

Peering is great for **a few** VPCs. For dozens → Transit Gateway.

---

## 2.13 Transit Gateway (TGW)

**TGW** = a regional **cloud router / hub** that connects many VPCs and on-prem networks in **hub-and-spoke** style.

```
          ┌── VPC-A
          ├── VPC-B
   [ TGW ]├── VPC-C
          ├── VPN (on-prem)
          └── Direct Connect Gateway
```

- **Transitive routing** ✅ (A can talk to C via TGW).
- Attachments: VPC, VPN, Direct Connect Gateway, TGW Peering (inter-region), Connect (SD-WAN).
- **TGW route tables** let you segment (e.g., prod can't talk to dev).
- Supports **ECMP** (multiple VPN tunnels to increase bandwidth beyond 1.25 Gbps per tunnel).
- Can be shared across accounts using **AWS RAM**.
- Costs: per attachment-hour + per GB processed.

### Peering vs TGW

| | Peering | Transit Gateway |
|---|---|---|
| Topology | 1:1 mesh | Hub-and-spoke |
| Transitive | ❌ | ✅ |
| Scale | Painful beyond ~5–10 VPCs (N×(N−1)/2 connections) | Thousands of attachments |
| Cost | Only data transfer | Attachment + per-GB fee |
| Latency / bandwidth | Lowest | Slight extra hop; high bandwidth |
| Best for | Few VPCs, simple, cheap | Many VPCs, on-prem integration |

---

## 2.14 Connecting to On-Premises

### Site-to-Site VPN

- **IPsec** tunnels over the **public internet**, encrypted.
- Components:
  - **Virtual Private Gateway (VGW):** AWS-side VPN endpoint attached to **one VPC** (or use a **Transit Gateway** as the AWS side).
  - **Customer Gateway (CGW):** a resource in AWS describing **your on-prem device** (public IP, BGP ASN).
- Each VPN connection has **2 tunnels** (two AZs) for redundancy; each up to **~1.25 Gbps**.
- Routing: **static** or **dynamic (BGP)**.
- Setup time: **minutes**. Cost: low.
- Add **route propagation** on the VPC route table to learn on-prem routes.

### AWS Client VPN
Managed **OpenVPN-based** remote-access VPN for **individual users** (laptops) to reach VPC / on-prem.

### AWS Direct Connect (DX)

- **Dedicated private physical fiber** from your data center/colo to AWS (not internet).
- Speeds: **Dedicated** 1 / 10 / 100 Gbps; **Hosted** 50 Mbps up to 25 Gbps (via partner).
- **Consistent low latency, high throughput**, lower data-out cost.
- **Not encrypted by default** → add **MACsec** or run a **Site-to-Site VPN over DX** for encryption.
- **Provisioning time: weeks to months.**
- **Virtual Interfaces (VIFs):**
  - **Private VIF** → access VPC resources (via VGW or DX Gateway)
  - **Public VIF** → access AWS public services (S3, DynamoDB) with public IPs
  - **Transit VIF** → connect to Transit Gateway
- **Direct Connect Gateway:** connect one DX to **multiple VPCs across regions/accounts**.
- For resiliency: **2 DX connections in different locations** (or DX + VPN backup).

### VPN vs Direct Connect (exam)

| Need | Choose |
|---|---|
| Quick setup, low cost, encryption by default | **Site-to-Site VPN** |
| Large steady data transfers (BFSI, migration), consistent latency | **Direct Connect** |
| Encrypted + private + dedicated | **DX + VPN (IPsec over DX)** |
| Backup for DX | **VPN** |
| Individual employees remote access | **Client VPN** |

> 🏦 BFSI note: Banks and insurers usually go **Direct Connect (primary) + VPN (backup)** into a **Transit Gateway** for regulated on-prem ↔ AWS data movement.

---

## 2.15 VPC Flow Logs

Capture **metadata about IP traffic** (not payload) going to/from network interfaces.

- Levels: **VPC**, **Subnet**, or **ENI**.
- Destinations: **CloudWatch Logs**, **S3**, **Kinesis Data Firehose**.
- Analyze with **Athena** (S3), **CloudWatch Logs Insights**, **OpenSearch**.
- Not real-time; records are aggregated in **1- or 10-minute** windows.
- Capture accepted, rejected, or all traffic.

### Default record format (v2)

```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
2 123456789012 eni-0abc 203.0.113.9 10.0.1.25 44321 22 6 20 4249 1700000000 1700000060 REJECT OK
```

Read it: *source 203.0.113.9 tried TCP (protocol 6) port 22 on 10.0.1.25 and was REJECTed.*
Protocol numbers: **6 = TCP, 17 = UDP, 1 = ICMP**.

### Troubleshooting trick
- **Inbound REJECT** and **outbound reply missing** → check SG/NACL.
- If **request ACCEPT but reply REJECT** → the **NACL** (stateless) is dropping replies — SGs are stateful and would not do this.

### Not logged
DNS traffic to Amazon's resolver, DHCP, instance metadata (`169.254.169.254`), Windows license activation, Time Sync (`169.254.169.123`), traffic to the VPC router's reserved IP.

---

## 2.16 Accessing Private Instances — Bastion vs SSM

### Bastion host (jump box)
A hardened EC2 in a **public subnet**; you SSH to it, then hop to private instances. Requirements: SG allowing SSH from **your IP only**; private instance SG allowing SSH from **bastion SG**.

### Better: AWS Systems Manager **Session Manager**
- No inbound ports, no public IP, no SSH keys.
- Requires: **SSM Agent** (preinstalled on Amazon Linux), **IAM instance profile** with `AmazonSSMManagedInstanceCore`, and a network path to SSM — via **NAT** *or* three **interface endpoints**: `ssm`, `ssmmessages`, `ec2messages`.
- Fully auditable (CloudTrail + session logs to S3/CloudWatch).

### EC2 Instance Connect Endpoint
Lets you SSH/RDP into private instances through the console/CLI without a bastion or public IP.

---

## 2.17 Other VPC Features to Recognize

| Feature | One-line purpose |
|---|---|
| **VPC Sharing (RAM)** | Share subnets across accounts in the same AWS Organization (central network team owns VPC; app accounts launch resources in it) |
| **AWS PrivateLink** | Expose a service privately to other VPCs/accounts through interface endpoints (needs NLB/GWLB in front) |
| **Traffic Mirroring** | Copy ENI traffic to inspection tools (IDS) |
| **Reachability Analyzer** | Static analysis: "Can A reach B?" — tells you which SG/NACL/route blocks it (no packets sent) |
| **Network Access Analyzer** | Find unintended network access paths |
| **AWS Network Firewall** | Managed stateful/stateless firewall, IPS, domain filtering at VPC level |
| **VPC IPAM** | Plan, track, and enforce CIDR allocation across accounts/regions |
| **Prefix Lists** | Reusable named set of CIDRs for SG/route rules |
| **Elastic Fabric Adapter / Placement Groups** | HPC networking (not VPC core) |
| **VPC Lattice** | Service-to-service networking (app layer) |

---

## 2.18 Putting It All Together — A Packet's Journey

**Scenario:** A user in Mumbai opens `https://shop.example.com` served by an ALB + EC2 in a private subnet that reads from RDS.

```
1. User's browser → DNS (Route 53) → ALB public IP
2. Internet → Internet Gateway (attached to VPC)
3. Route table of public subnet: ALB is in public subnet ✔
4. NACL (public subnet, inbound 443 allowed) ✔
5. ALB SG (inbound 443 from 0.0.0.0/0) ✔
6. ALB forwards to EC2 private IP in private subnet (local route inside VPC) 
7. NACL (private subnet inbound from public subnet CIDR) ✔
8. EC2 SG (inbound 8080 from alb-sg) ✔
9. EC2 → RDS (3306): db-sg allows 3306 from app-sg ✔ (data subnet has no route to internet)
10. EC2 needs OS patches → private RT → NAT GW → IGW → internet
11. EC2 writes logs to S3 → private RT → S3 Gateway Endpoint (free, no NAT)
12. Response returns through the same path (stateful SGs auto-allow; NACL needs ephemeral ports outbound)
```

If you can narrate this from memory, you understand VPC.

---

# Part 3 — Hands-On Labs (Live Practice)

> We use **Mumbai (`ap-south-1`)**, AZs `ap-south-1a` and `ap-south-1b`. You can use any region — just change names.

## ⚠️ Before You Start — Cost & Safety

| Resource | Cost warning |
|---|---|
| VPC, subnets, IGW, route tables, SG, NACL, gateway endpoint | **Free** |
| EC2 `t3.micro` / `t2.micro` | Free tier eligible (check your account's plan) |
| **NAT Gateway** | **Charged per hour + per GB** — delete it right after Lab 3 |
| Public IPv4 / Elastic IP | **Charged per hour** |
| Interface endpoints, VPC peering cross-region, flow logs storage | Small charges |

✅ Set a **Budget alert** (Billing → Budgets → e.g., ₹300) before you begin.
✅ Use an **IAM user/role with admin permissions** (not root) for labs.
✅ Always run the **Cleanup (Lab 9)** at the end.

### What you need
- AWS account, AWS Console access
- (Optional) AWS CLI v2 configured: `aws configure` (region `ap-south-1`)
- An EC2 **key pair** (we'll create one) — or just use SSM/EC2 Instance Connect.

---

## 🧪 Lab 0 — Look at the Default VPC (5 min)

**Goal:** Understand what AWS gives you for free, so you can appreciate what we build.

1. Console → search **VPC** → **Your VPCs**.
2. Find the VPC with **Default VPC = Yes** (CIDR `172.31.0.0/16`).
3. Click **Subnets** → notice one subnet per AZ, each `/20`, "Auto-assign public IPv4" = Yes.
4. Click **Route tables** → open the main one → **Routes** tab → see `0.0.0.0/0 → igw-…`.
5. Click **Internet gateways** → attached to the default VPC.

**Observation:** Default VPC = everything is public. Great for demos, bad for production. Now let's build a proper one.

CLI equivalent:
```bash
aws ec2 describe-vpcs --filters Name=isDefault,Values=true --query "Vpcs[0].[VpcId,CidrBlock]" --output text
```

---

## 🧪 Lab 1 — Build a Custom VPC from Scratch (30 min)

**Target architecture**

```
                         Internet
                             │
                       ┌─────┴─────┐
                       │    IGW    │
                       └─────┬─────┘
 ┌───────────────────────────┼─────────────── VPC 10.0.0.0/16 (lab-vpc) ──┐
 │                           │                                            │
 │   AZ ap-south-1a                          AZ ap-south-1b               │
 │  ┌───────────────────────┐              ┌───────────────────────┐      │
 │  │ public-a 10.0.1.0/24  │              │ public-b 10.0.2.0/24  │      │
 │  │ (RT: 0.0.0.0/0→IGW)   │              │ (RT: 0.0.0.0/0→IGW)   │      │
 │  └───────────────────────┘              └───────────────────────┘      │
 │  ┌───────────────────────┐              ┌───────────────────────┐      │
 │  │ private-a 10.0.11.0/24│              │ private-b 10.0.12.0/24│      │
 │  │ (RT: local only)      │              │ (RT: local only)      │      │
 │  └───────────────────────┘              └───────────────────────┘      │
 └─────────────────────────────────────────────────────────────────────────┘
```

### Step 1 — Create the VPC

1. VPC console → **Your VPCs** → **Create VPC**.
2. Choose **VPC only** (we want to learn each piece; "VPC and more" automates it).
3. Fill:
   - Name tag: `lab-vpc`
   - IPv4 CIDR: **Manual input** → `10.0.0.0/16`
   - IPv6: No IPv6 CIDR block
   - Tenancy: Default
4. **Create VPC**.
5. Select `lab-vpc` → **Actions → Edit VPC settings** → tick **Enable DNS hostnames** → Save. (DNS resolution is already on.)

✔ **Check:** In **Route tables**, a **main route table** for `lab-vpc` now exists with only the `10.0.0.0/16 → local` route. In **Security groups**, a default SG exists. In **Network ACLs**, a default NACL exists. These three are auto-created with every VPC.

### Step 2 — Create 4 Subnets

VPC → **Subnets** → **Create subnet** → select `lab-vpc`. Add subnets (you can add multiple on one screen):

| Name | AZ | IPv4 CIDR |
|---|---|---|
| `public-a` | ap-south-1a | `10.0.1.0/24` |
| `public-b` | ap-south-1b | `10.0.2.0/24` |
| `private-a` | ap-south-1a | `10.0.11.0/24` |
| `private-b` | ap-south-1b | `10.0.12.0/24` |

**Create subnet.**

✔ **Check:** Open `public-a` → "Available IPv4 addresses" shows **251**. (256 − 5 reserved. Now you've verified the theory.)

> Try: Create a subnet `10.0.1.0/24` again — AWS rejects it (overlap). Create `192.168.0.0/24` — rejected (outside VPC CIDR).

Enable auto-assign public IP on the public subnets:
- Select `public-a` → **Actions → Edit subnet settings** → tick **Enable auto-assign public IPv4 address** → Save. Repeat for `public-b`.

### Step 3 — Internet Gateway

1. **Internet gateways** → **Create internet gateway** → name `lab-igw` → Create.
2. State shows **Detached**. **Actions → Attach to VPC** → `lab-vpc` → Attach.

### Step 4 — Public Route Table

1. **Route tables** → **Create route table** → name `public-rt`, VPC `lab-vpc` → Create.
2. Select `public-rt` → **Routes** tab → **Edit routes** → **Add route**: Destination `0.0.0.0/0`, Target **Internet Gateway** → `lab-igw` → Save.
3. **Subnet associations** tab → **Edit subnet associations** → tick `public-a` and `public-b` → Save.

### Step 5 — Private Route Table (explicit is better than implicit)

1. Create route table `private-rt`, VPC `lab-vpc`.
2. **Subnet associations** → tick `private-a`, `private-b` → Save.
3. Routes: only `10.0.0.0/16 → local` for now (we'll add NAT in Lab 3).

✔ **Check (this is the key learning):**
- `public-a` → route table has `0.0.0.0/0 → igw` ⇒ **public subnet**.
- `private-a` → no IGW route ⇒ **private subnet**.
- You just built public vs private using nothing but **route tables**.

### 🧠 Reflection questions
1. Which route table do `public-a`/`private-a` use *before* association? (Answer: the main route table.)
2. If you added `0.0.0.0/0 → igw` to the *main* route table, what would happen to every new subnet? (They'd silently become public — why we keep main private.)

---

## 🧪 Lab 2 — Launch a Public Web Server (20 min)

**Goal:** Prove internet connectivity through IGW, and see Security Groups in action.

### Step 1 — Create a Security Group

EC2 console (or VPC → Security groups) → **Create security group**:
- Name: `web-sg`; Description: `Allow HTTP and SSH`; VPC: `lab-vpc`
- **Inbound rules:**
  - HTTP, TCP 80, Source `0.0.0.0/0`
  - SSH, TCP 22, Source **My IP**
- **Outbound:** leave default (all)

### Step 2 — Launch EC2

EC2 → **Launch instance**:
- Name: `web-public-a`
- AMI: **Amazon Linux 2023**
- Instance type: `t3.micro` (or `t2.micro` if free-tier shown)
- Key pair: create new `lab-key` (download `.pem`) — or choose "Proceed without" and use **EC2 Instance Connect**
- **Network settings → Edit:**
  - VPC: `lab-vpc`
  - Subnet: `public-a`
  - Auto-assign public IP: **Enable**
  - Select existing security group: `web-sg`
- **Advanced details → User data:**

```bash
#!/bin/bash
dnf install -y httpd
systemctl enable --now httpd
echo "<h1>Hello from $(hostname -f) — PUBLIC subnet</h1>" > /var/www/html/index.html
```

- **Launch instance.**

### Step 3 — Test

1. Wait for **2/2 status checks**. Copy the **Public IPv4 address**.
2. Browser: `http://<public-ip>` → you should see the Hello page. 🎉
3. SSH (from your laptop terminal):
   ```bash
   chmod 400 lab-key.pem
   ssh -i lab-key.pem ec2-user@<public-ip>
   ```
4. Inside the instance run:
   ```bash
   ip addr show        # you see ONLY the private IP 10.0.1.x  ← IGW does the NAT!
   curl -s https://checkip.amazonaws.com   # shows the PUBLIC IP
   curl -s http://169.254.169.254/latest/meta-data/local-ipv4   # metadata (IMDSv2 may require a token)
   ```

✔ **Observation:** The OS only knows `10.0.1.x`. The public IP exists only at the IGW. 

### Step 4 — Break it on purpose (learn by failure) 🔨

Run each experiment, observe, then **fix it**:

| # | Break | Expected symptom | Why | Fix |
|---|---|---|---|---|
| 1 | Remove HTTP rule from `web-sg` | Browser times out | SG default deny inbound | Re-add port 80 |
| 2 | Edit `public-rt` → delete `0.0.0.0/0 → igw` | Browser times out, SSH fails | No route to IGW (now a private subnet!) | Re-add route |
| 3 | Detach IGW from VPC (must delete the route first) | Same | No gateway | Re-attach |
| 4 | Launch a second instance in `public-a` with **auto-assign public IP = Disable** | Can't reach it from internet | No public IP | Allocate/associate an **Elastic IP** |
| 5 | Stop/Start the first instance | Public IP **changes** | Auto-assigned IPs aren't static | Use EIP if you need a fixed IP |

> Experiment 4 teaches the "four conditions" rule from section 2.2. **This is the exact interview question: "My EC2 is in a public subnet but I can't reach it from internet. Why?"**

---

## 🧪 Lab 3 — Private Instance + NAT Gateway + SSM (30 min)

**Goal:** Private server with no public IP that can still reach the internet (outbound only).

### Step 1 — Security group for the private instance

Create `app-sg` in `lab-vpc`:
- Inbound: HTTP 80 from **`web-sg`** (SG reference) — for later tests
- (No SSH rule needed — we'll use SSM)
- Outbound: all (default)

### Step 2 — IAM role for SSM

IAM → Roles → **Create role** → AWS service → **EC2** → attach policy **`AmazonSSMManagedInstanceCore`** → name `lab-ssm-role`. (Creates an instance profile automatically.)

### Step 3 — Launch private EC2

- Name: `app-private-a`, AMI: Amazon Linux 2023, `t3.micro`
- Network: `lab-vpc`, subnet **`private-a`**, auto-assign public IP **Disable**, SG `app-sg`
- Advanced → **IAM instance profile:** `lab-ssm-role`
- No key pair needed.

### Step 4 — Prove it has no internet (yet)

Wait ~2 min, then Systems Manager → **Fleet Manager / Session Manager**: the instance **will NOT appear** (SSM agent can't reach SSM service — no internet, no endpoints). 

This is expected! It proves the private subnet is truly isolated.

### Step 5 — Create the NAT Gateway

VPC → **NAT gateways** → **Create NAT gateway**:
- Name: `lab-nat-a`
- Subnet: **`public-a`** ← ⚠️ MUST be a *public* subnet (most common mistake)
- Connectivity type: **Public**
- **Allocate Elastic IP**
- Create. Wait until state = **Available** (~2 min).

### Step 6 — Route private subnets via NAT

Route tables → `private-rt` → Routes → Edit → Add route: `0.0.0.0/0` → **NAT Gateway** → `lab-nat-a` → Save.

### Step 7 — Test via Session Manager

1. Wait 2–5 minutes. Systems Manager → **Session Manager** → **Start session** → choose `app-private-a` → Start.
   (Or EC2 → select instance → Connect → **Session Manager** tab.)
2. Run:
   ```bash
   curl -s https://checkip.amazonaws.com    # prints the NAT Gateway's Elastic IP!
   sudo dnf -y update                      # works → proves outbound internet via NAT
   ip addr                                 # only 10.0.11.x
   ```
3. From **your laptop**, try to reach `10.0.11.x` — impossible (private).

✔ **Observation:** Outbound ✅ works. Inbound from internet ❌ impossible. The outbound IP seen by the world = the NAT's EIP.

### Step 8 — Test private-to-public communication inside VPC

From `app-private-a` (Session Manager), curl the public web server's **private IP**:

```bash
curl http://10.0.1.<x>
```

It works because `local` route lets all subnets in a VPC talk — and `web-sg` allows port 80 from `0.0.0.0/0`.

### 🧠 Questions
- What happens if NAT Gateway AZ fails? (Private subnets in that AZ lose internet. Solution: one NAT per AZ.)
- Why can't `private-a` ping the internet without NAT? (No route.)

> 💸 **Delete the NAT Gateway and release its EIP** when finished with this lab if you won't continue immediately — or complete Labs 4–5 quickly and run cleanup.

---

## 🧪 Lab 4 — NACL Experiment: Stateless & Deny Rules (20 min)

**Goal:** See stateless behavior and block an IP.

### Step 1 — Look at the default NACL
VPC → **Network ACLs** → the default NACL of `lab-vpc` → Inbound: `100 ALLOW all`, `* DENY all`. Same for outbound. Associated with all 4 subnets (default).

### Step 2 — Create a custom NACL

1. **Create network ACL**: name `public-nacl`, VPC `lab-vpc`.
2. Open it → **Inbound rules** → **only the `*` DENY** (custom NACLs start with deny-all).
3. **Subnet associations** → associate `public-a`.
4. **Immediately test** the web page → **times out** (everything denied).

### Step 3 — Allow only inbound HTTP (and see replies fail)

Add **inbound** rule: `100`, HTTP (80), TCP, `0.0.0.0/0`, ALLOW.
Browse the page → **still times out!** 

**Why?** Request is allowed inbound, but the **reply** (to ephemeral port) is blocked by outbound `*` DENY — NACLs are **stateless**.

### Step 4 — Fix with ephemeral outbound rule

Add **outbound** rule: `100`, Custom TCP, ports **`1024-65535`**, destination `0.0.0.0/0`, ALLOW.
Browse → ✅ the page loads.

### Step 5 — Block a specific IP

1. Find your public IP: `https://checkip.amazonaws.com`.
2. Add inbound rule **`90`**: All traffic, source `<your-ip>/32`, **DENY**.
3. Refresh page → ❌ blocked (rule 90 is evaluated before rule 100).
4. Change rule 90 to number **`110`** → refresh → ✅ works again (rule 100 allowed first → first match wins).

✔ **Observation:** Rule number order matters. That's the NACL lesson.

### Step 6 — Restore
Associate `public-a` back to the **default NACL** (or add SSH rule 120 + keep as is). Delete `public-nacl` later in cleanup.

---

## 🧪 Lab 5 — S3 Gateway Endpoint (20 min)

**Goal:** Private instance accesses S3 **without** NAT/internet.

### Step 1 — Prepare
- Create an S3 bucket: `lab-vpc-<yourname>-<random>` (globally unique).
- Upload a small `hello.txt`.
- Attach S3 read permission to `lab-ssm-role` (add `AmazonS3ReadOnlyAccess`).

### Step 2 — Remove the NAT route (so internet is gone)
`private-rt` → delete `0.0.0.0/0 → nat`. Now the private instance has **no internet**.

### Step 3 — Prove S3 is unreachable
Session Manager session — **note:** after removing NAT, the SSM session might die/not start (SSM also needs a path!). 
- **Option A (simplest):** Open the session first, *then* remove the NAT route; the existing session may stay alive briefly. 
- **Option B (proper):** Create the three SSM interface endpoints (`ssm`, `ssmmessages`, `ec2messages`) in private subnets with an SG allowing 443 from `10.0.0.0/16`, enabling **Private DNS**. Now SSM works with no internet. 

Then:
```bash
aws s3 ls s3://<bucket> --region ap-south-1     # hangs/timeouts → no path to S3
```
(Press Ctrl+C after ~10 sec.)

### Step 4 — Create the Gateway Endpoint

VPC → **Endpoints** → **Create endpoint**:
- Name: `lab-s3-gw`
- Type: **AWS services**
- Service: `com.amazonaws.ap-south-1.s3` — choose **Type = Gateway**
- VPC: `lab-vpc`
- Route tables: tick **`private-rt`**
- Policy: Full access (for now)
- Create.

### Step 5 — Observe the magic
1. Open `private-rt` → Routes → a new route appeared: destination **`pl-xxxxxxxx` (com.amazonaws.ap-south-1.s3)** → target **`vpce-xxxx`**. You did not add it manually — the endpoint did.
2. In the instance:
   ```bash
   aws s3 ls s3://<bucket> --region ap-south-1
   aws s3 cp s3://<bucket>/hello.txt .
   ```
   ✅ Works — **with no NAT, no IGW, no public IP.**
3. Try another service (e.g., `aws sqs list-queues`) → fails (no endpoint for SQS) — shows endpoint is service-specific.

### Step 6 — Restrict with an endpoint policy
Edit the endpoint policy to allow only your bucket (see JSON in section 2.11). Try listing a *different* bucket → **AccessDenied**. 

✔ **You've just implemented the standard BFSI "data never leaves AWS private network" pattern.**

---

## 🧪 Lab 6 — VPC Peering (25 min)

**Goal:** Connect two VPCs and understand non-transitivity.

### Step 1 — Create a second VPC
- `lab-vpc-2`, CIDR **`10.1.0.0/16`** (must not overlap `10.0.0.0/16`!)
- One subnet `b-private` `10.1.1.0/24` in `ap-south-1a`, own route table association (default main is fine).
- Launch EC2 `peer-test` in `b-private` (no public IP) with SSM role; SG `peer-sg` allowing **ICMP (all) and TCP 22 from `10.0.0.0/16`**.
  *(SSM access to this instance needs NAT/endpoints too — alternative: use EC2 Instance Connect Endpoint, or test from the other side.)*

### Step 2 — Request peering
VPC → **Peering connections** → **Create peering connection**:
- Requester: `lab-vpc`; Accepter: **My account → This Region → `lab-vpc-2`**
- Create → status **Pending acceptance**.
- **Actions → Accept request.** Status → Active.

### Step 3 — Update route tables (both sides!)
- `lab-vpc` → `private-rt` → add: `10.1.0.0/16 → pcx-xxxx`
- `lab-vpc-2` → its route table → add: `10.0.0.0/16 → pcx-xxxx`

### Step 4 — Test
From `app-private-a` (in lab-vpc):
```bash
ping -c 3 10.1.1.<ip-of-peer-test>
```
✅ replies. (Remove the route on one side → ping fails: proof both sides matter. Remove the SG ICMP rule → fails: proof SG matters.)

### Step 5 — Prove non-transitivity (thought experiment / optional)
Create `lab-vpc-3` (`10.2.0.0/16`) and peer it with `lab-vpc-2` only. From `lab-vpc` you **cannot** reach `10.2.x.x` even though `lab-vpc ↔ lab-vpc-2 ↔ lab-vpc-3`. Even if you add routes, peering won't forward — you need direct peering or TGW.

---

## 🧪 Lab 7 — VPC Flow Logs (20 min)

**Goal:** See accepted/rejected traffic in logs.

### Step 1 — Create the flow log
VPC → select `lab-vpc` → **Flow logs** tab → **Create flow log**:
- Filter: **All**
- Max aggregation interval: **1 minute**
- Destination: **Send to CloudWatch Logs** → create log group `/vpc/lab-vpc-flow` → IAM role: **Create new** (the wizard can create it)
- Format: AWS default
- Create.

### Step 2 — Generate traffic
- Browse the public web server page a few times.
- Try `ssh` to the public instance from a place whose IP is not allowed (or just run a port scan tool is **not** recommended; simply try port 8080 `curl http://<public-ip>:8080` → will be REJECTED).

### Step 3 — Read logs
CloudWatch → **Log groups** → `/vpc/lab-vpc-flow` → open a log stream (one per ENI). Wait ~2–5 minutes for data.

**CloudWatch Logs Insights query — top rejected sources:**
```
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT"
| stats count(*) as attempts by srcAddr, dstPort
| sort attempts desc
| limit 20
```

You will likely see random internet scanners hitting port 22/23/3389 — that's real life! 🌍

### Step 4 (BFSI/DE style) — Flow logs to S3 + Athena
Create another flow log with destination S3 (`arn:aws:s3:::<bucket>/flowlogs/`). Later, create an Athena table over it and query large histories cheaply — the standard approach for audit/forensics.

---

## 🧪 Lab 8 — Do the Entire Setup via AWS CLI (Infrastructure as commands) (30 min)

This reproduces Labs 1–3 in a script. Run in **CloudShell** (console top bar) or locally. Save as `build-vpc.sh`.

```bash
#!/bin/bash
set -euo pipefail
export AWS_REGION=ap-south-1
AZ1=ap-south-1a
AZ2=ap-south-1b

# ---------- 1. VPC ----------
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=cli-vpc}]' \
  --query Vpc.VpcId --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
echo "VPC: $VPC_ID"

# ---------- 2. Subnets ----------
mk_subnet () {  # $1=cidr $2=az $3=name
  aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block $1 --availability-zone $2 \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$3}]" \
    --query Subnet.SubnetId --output text
}
PUB_A=$(mk_subnet 10.0.1.0/24  $AZ1 cli-public-a)
PUB_B=$(mk_subnet 10.0.2.0/24  $AZ2 cli-public-b)
PRI_A=$(mk_subnet 10.0.11.0/24 $AZ1 cli-private-a)
PRI_B=$(mk_subnet 10.0.12.0/24 $AZ2 cli-private-b)
aws ec2 modify-subnet-attribute --subnet-id $PUB_A --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $PUB_B --map-public-ip-on-launch

# ---------- 3. Internet Gateway ----------
IGW_ID=$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# ---------- 4. Route tables ----------
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $PUB_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_A
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_B

PRI_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
aws ec2 associate-route-table --route-table-id $PRI_RT --subnet-id $PRI_A
aws ec2 associate-route-table --route-table-id $PRI_RT --subnet-id $PRI_B

# ---------- 5. Security group ----------
WEB_SG=$(aws ec2 create-security-group --group-name cli-web-sg --description "web" \
  --vpc-id $VPC_ID --query GroupId --output text)
MYIP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 22 --cidr ${MYIP}/32

# ---------- 6. EC2 (public web server) ----------
AMI=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query Parameter.Value --output text)

cat > userdata.sh <<'EOF'
#!/bin/bash
dnf install -y httpd
systemctl enable --now httpd
echo "<h1>Hello from CLI-built VPC</h1>" > /var/www/html/index.html
EOF

INSTANCE_ID=$(aws ec2 run-instances --image-id $AMI --instance-type t3.micro \
  --subnet-id $PUB_A --security-group-ids $WEB_SG \
  --user-data file://userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=cli-web}]' \
  --query 'Instances[0].InstanceId' --output text)
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "Open: http://$PUBLIC_IP"

# ---------- 7. (Optional) NAT Gateway — costs money! ----------
# EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query AllocationId --output text)
# NAT_ID=$(aws ec2 create-nat-gateway --subnet-id $PUB_A --allocation-id $EIP_ALLOC \
#          --query NatGateway.NatGatewayId --output text)
# aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_ID
# aws ec2 create-route --route-table-id $PRI_RT --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_ID

# ---------- 8. S3 gateway endpoint (free) ----------
aws ec2 create-vpc-endpoint --vpc-id $VPC_ID \
  --service-name com.amazonaws.$AWS_REGION.s3 --route-table-ids $PRI_RT

# ---------- Save IDs for cleanup ----------
cat > ids.env <<EOT
VPC_ID=$VPC_ID
PUB_A=$PUB_A
PUB_B=$PUB_B
PRI_A=$PRI_A
PRI_B=$PRI_B
IGW_ID=$IGW_ID
PUB_RT=$PUB_RT
PRI_RT=$PRI_RT
WEB_SG=$WEB_SG
INSTANCE_ID=$INSTANCE_ID
EOT
echo "Done. IDs saved in ids.env"
```

**Useful inspection commands:**

```bash
aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID \
  --query "Subnets[].[Tags[?Key=='Name']|[0].Value,CidrBlock,AvailabilityZone,AvailableIpAddressCount]" --output table

aws ec2 describe-route-tables --route-table-ids $PUB_RT --query "RouteTables[0].Routes" --output table

aws ec2 describe-security-groups --group-ids $WEB_SG --query "SecurityGroups[0].IpPermissions"
```

**Peering via CLI:**
```bash
PCX=$(aws ec2 create-vpc-peering-connection --vpc-id $VPC_ID --peer-vpc-id $VPC2_ID \
      --query VpcPeeringConnection.VpcPeeringConnectionId --output text)
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PCX
aws ec2 create-route --route-table-id $PRI_RT --destination-cidr-block 10.1.0.0/16 --vpc-peering-connection-id $PCX
```

**Flow log via CLI:**
```bash
aws ec2 create-flow-logs --resource-type VPC --resource-ids $VPC_ID --traffic-type ALL \
  --log-destination-type s3 --log-destination arn:aws:s3:::<your-bucket>/flowlogs/
```

**Reachability Analyzer (great practical tool):**
Console → VPC → **Reachability Analyzer** → Create and analyze path: Source = your private instance ENI, Destination = the web server ENI, protocol TCP, port 80 → **Run**. It tells you "Reachable" or points to the exact blocking SG/NACL/route.

---

## 🧪 Lab 9 — Cleanup (MANDATORY, 10 min)

Delete in **this order** (dependencies!):

1. **Terminate EC2 instances** (wait until terminated).
2. **Delete NAT Gateway** → wait until **Deleted** → **Release Elastic IP**.
3. Delete **VPC endpoints** (interface endpoints cost per hour).
4. Delete **VPC peering connections**.
5. Delete **Flow logs** (and the CloudWatch log group / S3 data if not needed).
6. **Detach** then **delete Internet Gateway**.
7. Delete **subnets**.
8. Delete **custom route tables** (disassociate subnets first; the main RT can't be deleted separately).
9. Delete **custom NACLs** and **custom security groups** (default ones go with the VPC).
10. **Delete VPC.**

CLI cleanup sketch (source `ids.env` first):
```bash
source ids.env
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID
# (delete NAT GW / release EIP / delete endpoints here if you created them)
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
for s in $PUB_A $PUB_B $PRI_A $PRI_B; do aws ec2 delete-subnet --subnet-id $s; done
aws ec2 delete-route-table --route-table-id $PUB_RT
aws ec2 delete-route-table --route-table-id $PRI_RT
aws ec2 delete-security-group --group-id $WEB_SG
aws ec2 delete-vpc --vpc-id $VPC_ID
```
> If a delete fails with `DependencyViolation`, something still uses it (ENI, endpoint, NAT GW, SG referenced by another SG). Find & delete it first. This error is **extremely** common.

✅ **Verify:** Billing → Cost Explorer next day. Check **Elastic IPs** and **NAT gateways** pages are empty.

---

## 🧪 Bonus Lab — "VPC and more" wizard (5 min)
Create VPC → choose **VPC and more** → 2 AZs, 2 public + 2 private subnets, **NAT gateways: In 1 AZ**, **VPC endpoints: S3 Gateway**. Preview shows the whole diagram. This builds in 1 click what you did manually. Now you can *explain* every item in that preview — that's the point of the earlier labs. Delete it after.

---

# Part 4 — Real-World Architecture Patterns

## 4.1 Standard 3-Tier Web App (HA)

```
                      Route 53
                         │
                 ┌───────┴────────┐
                 │  ALB (public)  │  ← public subnets in 2 AZs (SG: 443 from internet)
                 └───────┬────────┘
              ┌──────────┴──────────┐
        ┌─────┴─────┐         ┌─────┴─────┐
        │ EC2 (AZ-a)│         │ EC2 (AZ-b)│  ← private app subnets (SG: from alb-sg)
        └─────┬─────┘         └─────┬─────┘
              └──────────┬──────────┘
                  ┌──────┴───────┐
                  │ RDS Multi-AZ │  ← private data subnets (SG: 3306 from app-sg), no internet route
                  └──────────────┘
   NAT GW per AZ in public subnets for outbound patching
   S3 Gateway Endpoint in private route tables
```

Key points: ALB needs **subnets in ≥ 2 AZs**; RDS needs a **DB subnet group** with subnets in **≥ 2 AZs**.

## 4.2 Hub-and-Spoke with Transit Gateway (Enterprise / multi-account)

```
 Prod VPC ─┐
 Dev VPC  ─┤
 Shared    ├── Transit Gateway ── Direct Connect GW ── On-prem DC / Core Banking
 Services ─┤                    └─ Site-to-Site VPN (backup)
 Inspection VPC (Network Firewall) ─┘
```
Use **TGW route tables** to isolate prod from dev; send internet-bound traffic through a central **egress/inspection VPC**.

## 4.3 Data Engineering on VPC (very relevant for DEA-C01 and BFSI data platforms)

| Service | VPC behavior to remember |
|---|---|
| **Amazon S3** | Regional public service. From private subnets use **S3 Gateway Endpoint**; add bucket policy with `aws:SourceVpce` to force private access. |
| **AWS Glue** | Glue jobs run in AWS-managed VPC. To access **JDBC sources in your VPC** (RDS, Redshift, on-prem via DX), create a **Glue Connection** specifying VPC/subnet/SG. The SG must have a **self-referencing inbound rule (all TCP)** so Glue workers can talk to each other. The subnet needs a route to **S3 (Gateway Endpoint)** — and NAT or interface endpoints for other services (e.g., Glue, CloudWatch, KMS). |
| **Amazon Redshift** | Cluster lives in a **cluster subnet group** (≥ 1 subnets). **Enhanced VPC Routing** forces `COPY`/`UNLOAD` traffic through your VPC (so endpoints, NACLs, flow logs apply) instead of the public internet. Publicly accessible = false for private data warehouses. Port **5439**. |
| **Amazon RDS / Aurora / Oracle** | **DB subnet group** with subnets in ≥ 2 AZs. Keep in private/data subnets. SG: allow DB port (1521 Oracle, 3306 MySQL, 5432 Postgres) only from app/ETL SG. |
| **Amazon MSK (Kafka)** | Brokers are placed in **your VPC subnets** (one per AZ, typically 3). Clients must be in the same VPC, peered VPC, via TGW/VPN/DX, or use **MSK multi-VPC private connectivity** (PrivateLink-based). SG allows 9092/9094/9096/9098. |
| **Amazon EMR / EMR Serverless** | Launch in private subnets; need S3 gateway endpoint; may need interface endpoints (STS, Glue, CloudWatch Logs) or NAT. |
| **MWAA (Managed Airflow)** | Requires a VPC with **two private subnets in different AZs**; needs NAT or VPC endpoints for S3, SQS, CloudWatch, ECR, KMS, etc. Web server access mode: public or private. |
| **AWS Lambda in a VPC** | Lambda attaches ENIs in your subnets to reach private resources (RDS). To reach the internet or other AWS APIs it needs **NAT** (or interface endpoints). **Lambda in a *public* subnet does NOT get internet** (no public IP on its ENI). |
| **Kinesis / SQS / SNS / Secrets Manager / KMS** | Use **Interface Endpoints** for fully private access. |
| **Athena / QuickSight** | QuickSight can connect to private data sources by creating a **VPC connection** (ENIs in your subnets). |
| **Snowflake / Databricks on AWS** | Use **PrivateLink** endpoints for private connectivity from your VPC (control plane/data plane); Databricks workspaces deploy into **your VPC** (customer-managed VPC). |
| **DMS** | Replication instance placed in **your VPC subnet group**; security groups must allow source and target DB ports. On-prem source via VPN/DX. |

### Compliance mindset (BFSI / IRDAI / DPDP)
- Keep data stores in **private/data subnets with no route to IGW**.
- Use **VPC endpoints + endpoint policies + bucket policies** → data never traverses the public internet.
- Use **encryption in transit** (TLS, VPN/MACsec) and at rest (KMS).
- Enable **VPC Flow Logs → S3** with retention for audit; **CloudTrail** for API calls.
- **Data residency:** deploy VPCs in India regions (e.g., `ap-south-1` Mumbai, `ap-south-2` Hyderabad) and avoid cross-region replication of regulated data unless approved.
- Prefer **Direct Connect + VPN backup** for on-prem core systems.
- Segregate environments (prod/non-prod) using **separate accounts + TGW route table isolation**.

## 4.4 Bastion-less Admin Access Pattern
Private instances + **SSM Session Manager** + interface endpoints (`ssm`, `ssmmessages`, `ec2messages`) + session logging to S3/CloudWatch. No public IPs, no port 22, full audit.

## 4.5 Centralized Egress
All private subnets across VPCs → TGW → **egress VPC** with NAT GW (or Network Firewall) → IGW. Saves NAT cost, centralizes inspection.

---

# Part 5 — Exam Cheat Sheet

## 5.1 The 25 Facts You Must Know

1. VPC = **regional**; Subnet = **one AZ** only.
2. VPC CIDR **/16 to /28**. Can add secondary CIDRs; can't edit/shrink primary.
3. **5 reserved IPs** per subnet (.0 .1 .2 .3 and last).
4. Default VPC: `172.31.0.0/16`, public subnets, IGW attached.
5. Public subnet = route to **IGW**. Nothing else defines it.
6. Internet access needs: **IGW + route + public IP + SG/NACL**.
7. **IGW** is HA, horizontally scaled, free; one per VPC.
8. **NAT GW** lives in a **public** subnet, needs **EIP**, zonal → **one per AZ** for HA, **no SG**.
9. NAT Instance → **disable source/destination check**.
10. **SG**: stateful, allow-only, instance-level; default inbound deny / outbound allow.
11. **NACL**: stateless, allow+deny, subnet-level, numbered rules, first match wins; default NACL allows all; custom denies all.
12. To **block an IP** → **NACL** (not SG).
13. NACL replies need **ephemeral ports 1024–65535**.
14. SG can reference another SG; NACL cannot.
15. Route table: **local route** always present; **longest prefix match**.
16. A subnet ↔ **one** route table; a route table ↔ **many** subnets.
17. **Gateway endpoint** → **S3 & DynamoDB only**, free, route-table based, not reachable from on-prem/peered.
18. **Interface endpoint** → ENI, PrivateLink, costs, SG, works from on-prem; most services.
19. **Peering**: no overlapping CIDR, **non-transitive**, update routes both sides.
20. **Transit Gateway**: hub-and-spoke, transitive, scales to thousands of attachments.
21. **VPN**: IPsec over internet, 2 tunnels, VGW + CGW, minutes to set up.
22. **Direct Connect**: dedicated, private, weeks to provision, **not encrypted by default**; DX Gateway for multi-VPC/region.
23. **Egress-only IGW** = IPv6 outbound only; NAT GW = IPv4.
24. **Flow Logs** capture metadata at VPC/subnet/ENI → CloudWatch/S3/Firehose; not payload; not real-time.
25. **enableDnsSupport + enableDnsHostnames** are needed for private DNS on endpoints & private hosted zones.

## 5.2 "If you see X in the question → think Y"

| Question clue | Answer |
|---|---|
| "Private instances need to download patches, no inbound from internet" | **NAT Gateway** in public subnet |
| "Access S3 from private subnet without internet / lowest cost" | **S3 Gateway Endpoint** |
| "Access S3 from on-premises privately" | **Interface endpoint for S3 over DX/VPN** (or public VIF) |
| "Block a malicious IP address" | **NACL deny rule** (or WAF for web layer) |
| "Allow DB access only from app servers" | **SG referencing app SG** |
| "Connect 50+ VPCs and on-prem" | **Transit Gateway** |
| "Connect two VPCs, simple & cheapest" | **VPC Peering** |
| "Overlapping CIDRs but need connectivity to a service" | **PrivateLink** (not peering) |
| "Encrypted, quick connectivity to on-prem" | **Site-to-Site VPN** |
| "Consistent, high-bandwidth, private link to on-prem" | **Direct Connect** |
| "DX + encryption" | **VPN over DX** or **MACsec** |
| "IPv6 instances outbound only" | **Egress-only IGW** |
| "Analyze who accessed/was rejected" | **VPC Flow Logs** |
| "Why can't A reach B? (no packet sending)" | **Reachability Analyzer** |
| "Share subnets across accounts" | **RAM (VPC sharing)** |
| "Admin access to private EC2, no SSH/bastion" | **SSM Session Manager** |
| "Stateless firewall" | **NACL** |
| "Inspect/filter traffic with IPS, domain filtering" | **AWS Network Firewall** |
| "Expose my SaaS to customers privately" | **PrivateLink (NLB + endpoint service)** |
| "Remote employees' laptops to VPC" | **Client VPN** |
| "NAT GW in AZ-a fails, instances in AZ-a lose internet" | **NAT GW per AZ** |
| "Lambda in VPC can't call external API" | **Add NAT GW** (Lambda is not in public subnet-IGW path) |

## 5.3 Common Traps

- ❌ "A subnet is public if auto-assign public IP is on." → ✅ Public = route to IGW.
- ❌ "NAT Gateway in the private subnet." → ✅ Must be in **public** subnet.
- ❌ "One NAT GW is HA for all AZs." → ✅ It's zonal.
- ❌ "Peering is transitive / can use peer's NAT." → ✅ No.
- ❌ "Gateway endpoint works from on-prem." → ✅ No.
- ❌ "SG can deny." → ✅ Allow only.
- ❌ "NACL is stateful." → ✅ Stateless.
- ❌ "Deleting NACL rule order doesn't matter." → ✅ Lowest number first.
- ❌ "Instance has public IP configured in OS." → ✅ IGW does 1:1 NAT.
- ❌ "EIP is free." → ✅ All public IPv4 charged now.
- ❌ "Subnet can span AZs." → ✅ Never.

---

# Part 6 — Practice Questions with Explanations

> Format: Question → Answer → Why. Cover the answer first and try!

**Q1.** A company's EC2 instances in a private subnet need to download OS patches from the internet. No one from the internet should initiate connections. What should the solutions architect do?
**A:** Create a **NAT Gateway in a public subnet**, add `0.0.0.0/0 → NAT GW` in the private subnet's route table.
**Why:** NAT allows outbound-initiated traffic only. Placing NAT in a private subnet fails (it needs IGW route).

**Q2.** An EC2 instance in a public subnet cannot be reached from the internet. IGW is attached and the route table has `0.0.0.0/0 → IGW`. SG allows port 80. What else could be wrong?
**A:** The instance has **no public/Elastic IP**, or the **NACL** is blocking.
**Why:** Four conditions: IGW, route, public IP, SG/NACL.

**Q3.** You must block traffic from a specific malicious IP to all instances in a subnet. Which is the best option?
**A:** **NACL deny rule** with a lower number than the allow rules.
**Why:** SGs can't deny.

**Q4.** After creating a custom NACL, web users can send requests but receive no responses. Inbound rule allows 80/443. What's missing?
**A:** **Outbound rule for ephemeral ports (1024–65535)**.
**Why:** NACLs are stateless.

**Q5.** A company has 3 VPCs: A↔B and B↔C are peered. Can A reach C?
**A:** **No.** Peering is non-transitive. Create A↔C peering or use TGW.

**Q6.** Two VPCs have the same CIDR `10.0.0.0/16`. Can you peer them?
**A:** **No** — overlapping CIDRs are not allowed. Options: re-IP, or use **PrivateLink** for specific service access, or private NAT gateway design.

**Q7.** Private EC2 instances transfer terabytes to S3 daily through a NAT Gateway and the NAT bill is huge. How to reduce cost?
**A:** Create an **S3 Gateway VPC Endpoint** and add it to the private route tables.
**Why:** Gateway endpoints are free and bypass NAT data processing charges.

**Q8.** On-prem servers must access S3 privately over Direct Connect/VPN without public internet. Which endpoint?
**A:** **Interface VPC endpoint for S3** (PrivateLink).
**Why:** Gateway endpoints are not accessible from on-prem.

**Q9.** A database should accept connections only from application servers that scale in and out. Best approach?
**A:** DB SG inbound rule with **source = app servers' security group**.
**Why:** IPs change in Auto Scaling; SG reference stays correct.

**Q10.** A company needs a dedicated, consistent 10 Gbps private connection to AWS from a data center, and the data must also be encrypted. What?
**A:** **Direct Connect + Site-to-Site VPN over DX** (or MACsec on supported dedicated connections).
**Why:** DX isn't encrypted by default.

**Q11.** A company needs connectivity to AWS within hours (urgent migration), then will move to a dedicated line later. What now?
**A:** **Site-to-Site VPN** first.
**Why:** DX takes weeks.

**Q12.** A company has 40 VPCs across accounts and an on-prem DC. They want simple central routing. Which service?
**A:** **AWS Transit Gateway**.
**Why:** Peering mesh = 780 connections for 40 VPCs and still non-transitive.

**Q13.** You want to give your SaaS customers private access to your service running behind an NLB in your VPC, without peering or exposing CIDRs. Which?
**A:** **AWS PrivateLink** (VPC endpoint service). Customers create interface endpoints.

**Q14.** IPv6-enabled instances in private subnets need outbound internet only. Which gateway?
**A:** **Egress-Only Internet Gateway** (`::/0 → eigw`).

**Q15.** A NAT Gateway is deployed in a single AZ. The AZ fails and apps in other AZs lose internet. Why?
**A:** All private route tables point to the single AZ's NAT GW. Fix: **one NAT GW per AZ**, each private subnet routes to the NAT in its own AZ.

**Q16.** You see in Flow Logs: `... 203.0.113.9 10.0.1.25 44321 22 6 ... REJECT`. What happened?
**A:** TCP (6) connection from 203.0.113.9 to SSH port 22 was **rejected** by SG or NACL.

**Q17.** Flow logs show inbound request ACCEPT but the response REJECT. What's the cause?
**A:** **NACL** (stateless) outbound rule blocking return traffic. (SG would not do this.)

**Q18.** What is the minimum and maximum size of a VPC IPv4 CIDR block?
**A:** **/28 (16 IPs)** minimum; **/16 (65,536 IPs)** maximum.

**Q19.** How many usable IPs in a /27 subnet in AWS?
**A:** 32 − 5 = **27**.

**Q20.** An AWS Glue job must read from an RDS database in a private subnet. What do you create/configure?
**A:** A **Glue Connection** (VPC, subnet, SG). The SG needs a **self-referencing inbound rule**. The subnet needs **S3 endpoint** (and NAT/endpoints for other services).

**Q21.** A Redshift cluster does `COPY` from S3 and the company wants all traffic to stay in the VPC and be logged. Which feature?
**A:** **Enhanced VPC Routing** (plus S3 gateway endpoint).

**Q22.** A Lambda function in a VPC (placed in a **public subnet**) can't reach the internet. Why?
**A:** Lambda ENIs never get public IPs; need **private subnet + NAT GW**.

**Q23.** You need to SSH into private instances for audit-heavy BFSI environment with no open inbound ports and full session logs. Best option?
**A:** **SSM Session Manager** (with endpoints or NAT), logs to S3/CloudWatch.

**Q24.** You want to know why instance A cannot connect to instance B without generating traffic. Tool?
**A:** **VPC Reachability Analyzer**.

**Q25.** You changed a security group rule. Do you need to restart the instance?
**A:** **No**, effect is immediate; existing connections tracked continue.

**Q26.** A subnet's route table has no route to an IGW or NAT. Which statement is true?
**A:** It is a **private subnet** and instances can reach only VPC (local) and any configured endpoints/peering/VPN targets.

**Q27.** The CIDR of the VPC is 10.0.0.0/16. You try to create a route `10.0.0.0/16 → igw-xxx`. What happens?
**A:** **The route is rejected / has no effect.** The local route for the VPC CIDR always wins, so traffic to addresses inside the VPC stays inside the VPC and can't be redirected to an IGW or any other target.

**Q28.** You need an RDS Multi-AZ instance. What network prerequisite exists?
**A:** A **DB subnet group** with subnets in **at least 2 AZs**.

**Q29.** A company wants a central network team to own VPCs and subnets while application teams in different accounts launch EC2 into them. Which?
**A:** **VPC Sharing via AWS RAM** (AWS Organizations).

**Q30.** Which statement about Gateway endpoints is FALSE? (a) free (b) S3 & DynamoDB (c) uses route tables (d) reachable via VPN from on-prem
**A:** **(d)** is false. Gateway endpoints aren't accessible from on-prem or peered VPCs.

**Q31.** You must restrict an S3 bucket so it's only accessible from your VPC. How?
**A:** Use a **bucket policy with `aws:SourceVpce` (or `aws:SourceVpc`) condition**, with the S3 gateway endpoint in place.

**Q32.** How do you increase the IP capacity of a VPC that has run out of addresses?
**A:** **Associate a secondary CIDR block** with the VPC and create new subnets there.

---

# Part 7 — Troubleshooting Playbook

When "A can't connect to B", walk the path **in this order**. It works for 95% of cases:

```
1. DNS         → Does the name resolve? (nslookup / dig). enableDnsSupport on?
2. Route table → Does the SOURCE subnet have a route to the destination (IGW / NAT / pcx / tgw / local)?
                 Does the DESTINATION subnet have a route BACK (for peering/VPN/TGW both sides!)?
3. Public IP   → (Internet-facing) Does the instance have public/Elastic IP? IGW attached?
4. NACL        → Source subnet outbound + destination subnet inbound allow it?
                 Reply path: destination outbound + source inbound with ephemeral ports?
5. Security Group → Source SG outbound allows? Destination SG inbound allows (port, protocol, source)?
6. OS firewall → iptables / firewalld / Windows firewall? Is the app actually listening? (ss -tlnp)
7. App         → Correct port? Service running? Bound to 0.0.0.0 not 127.0.0.1?
```

### Symptom → Likely cause

| Symptom | Likely cause |
|---|---|
| **Timeout** (hangs) | SG/NACL silently dropping, or missing route |
| **Connection refused** (immediate) | Network is fine; **nothing listening** on that port / OS firewall |
| Works from one subnet, not another | Different NACL/route table |
| Private instance can't `yum/dnf update` | No NAT route; NAT in wrong subnet; NAT's subnet RT lacks IGW; SG outbound blocked |
| SSM Session Manager "instance not listed" | No IAM role; no path to SSM (NAT/endpoints); agent not running; endpoint SG blocks 443 |
| Peering up but ping fails | Route missing on one side; SG/NACL; overlapping CIDR |
| Can't delete VPC/subnet/SG | `DependencyViolation` — ENIs, NAT GW, endpoints, referenced SGs still exist |
| S3 via endpoint "AccessDenied" | Endpoint policy or bucket policy `SourceVpce` mismatch; IAM role lacks permission |
| Interface endpoint name resolves to public IP | **Private DNS** not enabled, or DNS attributes off |
| High NAT bill | S3/DynamoDB traffic via NAT → add gateway endpoints; cross-AZ NAT traffic |
| Lambda timeouts in VPC | No NAT/endpoint to reach dependent service |

---

# Part 8 — Final Revision Checklist

### Concepts (tick when you can explain out loud, without notes)

- [ ] What is CIDR, how to compute IPs from prefix
- [ ] Private IP ranges
- [ ] Stateful vs stateless
- [ ] Region vs AZ vs VPC vs Subnet scope
- [ ] What makes a subnet public
- [ ] 5 reserved IPs
- [ ] Route table, local route, longest prefix match
- [ ] IGW, public IP vs EIP, why OS doesn't see public IP
- [ ] NAT GW (placement, zonal, EIP, cost) vs NAT instance
- [ ] SG vs NACL (full table)
- [ ] Ephemeral ports
- [ ] Gateway vs Interface endpoints, PrivateLink
- [ ] Peering rules (non-transitive, no overlap, both-side routes)
- [ ] Transit Gateway use cases
- [ ] VPN vs Direct Connect vs DX+VPN; VGW, CGW, VIFs, DX Gateway
- [ ] Flow logs levels, destinations, record reading
- [ ] DNS attributes, DHCP option sets
- [ ] IPv6 and Egress-only IGW
- [ ] Bastion vs SSM Session Manager
- [ ] VPC behavior of Glue, Redshift, RDS, MSK, MWAA, Lambda

### Labs (tick when done)

- [ ] Lab 1 Custom VPC · [ ] Lab 2 Public web server · [ ] Lab 3 NAT + SSM
- [ ] Lab 4 NACL · [ ] Lab 5 S3 endpoint · [ ] Lab 6 Peering
- [ ] Lab 7 Flow logs · [ ] Lab 8 CLI script · [ ] Lab 9 Cleanup

### 60-second spoken summary (practice this for interviews)

> "A VPC is my private, regional virtual network in AWS, defined by a CIDR between /16 and /28. I split it into subnets, each in one AZ. A subnet is public only if its route table sends 0.0.0.0/0 to an Internet Gateway; otherwise it's private. Private subnets reach the internet through a NAT Gateway placed in a public subnet — one per AZ for HA. I secure with security groups — stateful, allow-only, at the instance level, referencing each other by SG — and NACLs — stateless, ordered allow/deny rules at the subnet level. For AWS services I use VPC endpoints: free gateway endpoints for S3 and DynamoDB, and PrivateLink interface endpoints for the rest. I connect VPCs with peering for simple cases or Transit Gateway at scale, and connect to on-prem with Site-to-Site VPN or Direct Connect. I monitor with Flow Logs and troubleshoot with Reachability Analyzer."

---

### 🔁 What to study next (natural continuation)
1. **Route 53** (private hosted zones, resolver) · 2. **Elastic Load Balancing** (ALB/NLB/GWLB inside VPC) · 3. **AWS Direct Connect & Transit Gateway deep dive** · 4. **Network Firewall / WAF / Shield** · 5. **IAM + KMS** for data-protection layers.

*End of notes. Build it, break it, fix it — that's how VPC sticks.* 🚀
