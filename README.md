<div align="center">

# 🌐 OSPF Distribute-List Lab — Filtering Odd & Even Routes with ACL Wildcard Masks

### OSPF Route Filtering (`distribute-list in`) | Standard ACL `0.0.14.0` | GNS3 + Cisco IOS

![Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![OSPF](https://img.shields.io/badge/Protocol-OSPF-green?style=for-the-badge)
![Filter](https://img.shields.io/badge/Feature-distribute--list%20in-orange?style=for-the-badge)
![ACL](https://img.shields.io/badge/ACL-Standard%20%2B%20Wildcard-red?style=for-the-badge)
![Routes](https://img.shields.io/badge/Route%20Type-O%20E2-blueviolet?style=for-the-badge)
![AD](https://img.shields.io/badge/Admin%20Distance-110-yellow?style=for-the-badge)
![Lab](https://img.shields.io/badge/Lab-GNS3-informational?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed-success?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA%20%E2%86%92%20CCNP-purple?style=for-the-badge)

*R1 injects a block of external routes into OSPF. On R2, a **distribute-list** with a single clever wildcard mask (`0.0.14.0`) lets through **only the odd-numbered networks** — then, with a second list, **only the even-numbered networks**.*

</div>

---

## 📑 Table of Contents

1. [Project Overview](#-project-overview)
2. [OSPF Introduction](#-ospf-introduction)
3. [OSPF Route Types](#-ospf-route-types)
4. [Route Filtering & Distribute-List](#-route-filtering--distribute-list)
5. [The Odd/Even Wildcard Trick](#-the-oddeven-wildcard-trick)
6. [Lab Topology](#-lab-topology)
7. [Addressing & Route Table](#-addressing--route-table)
8. [Configuration](#-configuration)
9. [Verification (Stage by Stage)](#-verification-stage-by-stage)
10. [Distribute-List Behavior & Limits](#-distribute-list-behavior--limits)
11. [Other Ways to Filter OSPF](#-other-ways-to-filter-ospf)
12. [Command Cheat Sheet](#-command-cheat-sheet)
13. [Key Takeaways](#-key-takeaways)
14. [Troubleshooting Checklist](#-troubleshooting-checklist)
15. [Skills Demonstrated](#-skills-demonstrated)
16. [Author](#-author)

---

## 🎯 Project Overview

| Item | Details |
|------|---------|
| **Lab Name** | OSPF Distribute-List — separating odd and even routes |
| **Platform** | GNS3 (Cisco IOS routers), accessed over Telnet |
| **Topology** | Two routers: **R1 (1.1.1.1)** ⇄ **R2 (1.1.1.2)** over FastEthernet0/0 |
| **Route source** | R1 creates **16 static routes to `Null0`** and injects them into OSPF |
| **Route type on R2** | **`O E2`** — OSPF external type 2, `[110/20]` |
| **Filter point** | **R2**, under `router ospf 1`, direction **`in`** |
| **Goal** | Control which OSPF routes R2 installs in its routing table |
| **Result** | **Odd** networks only → then **even** networks only |

### 🧪 The Experiment

| Stage | Action on R2 | R2 routing table shows |
|-------|-------------|------------------------|
| **1** | *(Expected baseline — all injected routes)* | Every external route from R1 |
| **2** | `access-list 1` + `distribute-list 1 in` | **Odd only:** `.1 .3 .5 .7 .9 .11 .13 .15` |
| **3** | `access-list 2` + `distribute-list 2 in` | **Even only:** `.2 .4 .6 .8 .10 .12 .14` |

---

## 📘 OSPF Introduction

**OSPF (Open Shortest Path First)** is a **link-state Interior Gateway Protocol (IGP)** defined in RFC 2328.

| Property | Description |
|----------|-------------|
| **Category** | IGP — link-state |
| **Algorithm** | **Dijkstra SPF** over a link-state database (LSDB) |
| **Metric** | **Cost** (based on interface bandwidth) |
| **Administrative Distance** | **110** |
| **Transport** | IP protocol **89** (multicast 224.0.0.5 / 224.0.0.6) |
| **Hierarchy** | Areas — **Area 0 (backbone)** is mandatory |
| **Neighbors** | Discovered automatically with Hello packets |
| **Updates** | Triggered LSAs; each router builds the same LSDB and runs SPF |

### 🔄 OSPF Neighbor States

```
Down → Init → 2-Way → ExStart → Exchange → Loading → Full
```

### 🧠 Why "link-state" matters for filtering

Every OSPF router in an area holds the **same LSDB** and calculates its own routes. That is why OSPF route filtering works differently from distance-vector protocols: you cannot simply stop a neighbor from *sending* routes — you can only decide which calculated routes get **installed** (see [Distribute-List Behavior & Limits](#-distribute-list-behavior--limits)).

---

## 🗂️ OSPF Route Types

| Code | Type | Metric behavior | Source |
|------|------|-----------------|--------|
| `O` | Intra-area | Cumulative cost | Same area |
| `O IA` | Inter-area | Cumulative cost | Another area |
| `O E1` | External type 1 | External metric **+ internal cost** | Redistributed |
| **`O E2`** ⭐ | **External type 2 (default)** | **External metric only** (does not grow) | **Redistributed** |
| `O N1/N2` | NSSA external | Like E1/E2 | NSSA area |

### 🔎 The routes in this lab

```
O E2  192.168.1.0/24 [110/20] via 1.1.1.1, 00:00:11, FastEthernet0/0
        │                │  │       │
        │                │  │       └── Next hop = R1
        │                │  └────────── Metric 20 (default external metric for redistributed routes)
        │                └───────────── Administrative Distance 110 (OSPF)
        └────────────────────────────── E2 = External Type 2 (redistributed into OSPF)
```

> 💡 `O E2` tells us R1 **redistributes** its static routes into OSPF. The default E2 metric is **20**, and it stays 20 no matter how many hops the route travels.

---

## 🚦 Route Filtering & Distribute-List

A **distribute-list** applies an **ACL (or prefix-list/route-map)** to routes entering or leaving a routing process.

| Item | Description |
|------|-------------|
| **Command** | `distribute-list <acl> in` (under `router ospf <pid>`) |
| **Direction** | `in` = filters routes **before they are installed** in the local routing table |
| **ACL type** | Standard ACL matches the **network address** of the route |
| **Implicit deny** | Any route **not permitted** by the ACL is blocked |
| **Scope** | **Local to this router only** |

### 🧭 How it flows

```
 R1 ──LSAs──►  R2 LSDB (all routes, unchanged)
                  │
                  ▼  SPF calculation
          ┌──────────────────┐
          │ distribute-list  │  ◄── ACL: permit / deny each prefix
          └────────┬─────────┘
                   ▼
            R2 routing table  (only permitted routes installed)
```

---

## 🧮 The Odd/Even Wildcard Trick

One ACL line can match **every odd** (or **every even**) third-octet value, using the wildcard mask **`0.0.14.0`**.

### 🔢 Binary breakdown of the third octet

Wildcard `14` = `00001110` → **bits 1, 2, 3 are "don't care"**; **bit 0** and **bits 4–7 must match**.

| Bit position | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
|---|---|---|---|---|---|---|---|---|
| **Wildcard 14** | 0 | 0 | 0 | 0 | 1 | 1 | 1 | 0 |
| Meaning | must match | must match | must match | must match | free | free | free | **must match** |

### 🟢 Odd networks — `permit 192.168.1.0 0.0.14.0`

Base third octet **1** = `00000001` → bit 0 must be **1**, bits 4–7 must be **0**.

| Third octet | Binary | Match? |
|-------------|--------|--------|
| 1 | 0000 0001 | ✅ |
| 3 | 0000 0011 | ✅ |
| 5 | 0000 0101 | ✅ |
| 7 | 0000 0111 | ✅ |
| 9 | 0000 1001 | ✅ |
| 11 | 0000 1011 | ✅ |
| 13 | 0000 1101 | ✅ |
| 15 | 0000 1111 | ✅ |
| 2, 4, 6 … 14 (even) | bit 0 = 0 | ❌ |
| 16 | 0001 0000 | ❌ (bit 4 set) |

### 🔵 Even networks — `permit 192.168.0.0 0.0.14.0`

Base third octet **0** = `00000000` → bit 0 must be **0**, bits 4–7 must be **0**.

| Third octet | Match? |
|-------------|--------|
| 0, 2, 4, 6, 8, 10, 12, 14 | ✅ |
| 1, 3, 5 … 15 (odd) | ❌ (bit 0 = 1) |
| 16 | ❌ (bit 4 set) |

> ⚠️ The wildcard only spans values **0–15**. A route such as `192.168.16.0` matches **neither** list. It would need its own line, e.g. `access-list 2 permit 192.168.16.0 0.0.0.0`.

---

## 🗺️ Lab Topology

```
 ┌──────────────────────┐        1.0.0.0 / 24         ┌──────────────────────┐
 │       Router1        │ Fa0/0 ───────────────── Fa0/0 │       Router2        │
 │         (R1)         │ 1.1.1.1                1.1.1.2 │         (R2)         │
 │                      │                              │                      │
 │  16 static routes    │   OSPF adjacency             │  distribute-list in  │
 │  → Null0             │   ─────────────────►         │  (ACL 1 → odd)       │
 │  redistributed       │   O E2 routes                │  (ACL 2 → even)      │
 │  into OSPF           │                              │                      │
 └──────────────────────┘                              └──────────────────────┘
```

### 🧱 Static routes on R1 (to Null0)

R1 creates a block of "fake" networks that point to **`Null0`** (a discard interface). This is a common way to generate many routes without needing real interfaces.

```bash
R1(config)# ip route 192.168.1.0  255.255.255.0 Null0
R1(config)# ip route 192.168.2.0  255.255.255.0 Null0
R1(config)# ip route 192.168.3.0  255.255.255.0 Null0
...
R1(config)# ip route 192.168.16.0 255.255.255.0 Null0
```

---

## 📋 Addressing & Route Table

| Device | Interface | IP Address | Mask |
|--------|-----------|------------|------|
| **R1** | FastEthernet0/0 | 1.1.1.1 | 255.255.255.0 |
| **R2** | FastEthernet0/0 | 1.1.1.2 | 255.255.255.0 |

### Routes injected by R1

| Set | Networks |
|-----|----------|
| 🟢 **Odd** | 192.168.1.0, .3.0, .5.0, .7.0, .9.0, .11.0, .13.0, .15.0 (/24 each) |
| 🔵 **Even** | 192.168.2.0, .4.0, .6.0, .8.0, .10.0, .12.0, .14.0, .16.0* (/24 each) |

\* *`192.168.16.0` is outside the `0.0.14.0` filter range (see the wildcard note above).*

---

## 🛠️ Configuration

### 🔹 R1 (route source) — reconstructed from the outputs

```bash
R1(config)# ip route 192.168.<1-16>.0 255.255.255.0 Null0   ! 16 static routes
R1(config)# router ospf 1
R1(config-router)# network 1.1.1.0 0.0.0.255 area 0
R1(config-router)# redistribute static subnets
```

> 📝 The R1 and OSPF adjacency configuration is not shown in the screenshots. `O E2` on R2 proves the static routes are redistributed into OSPF.

### 🔹 R2 — Step 1: Odd networks

```bash
R2(config)# access-list 1 permit 192.168.1.0 0.0.14.0
R2(config)# do write
Building configuration...
[OK]
R2(config)# router ospf 1
R2(config-router)# distribute-list 1 in
```

### 🔹 R2 — Step 2: Even networks

```bash
R2(config)# access-list 2 permit 192.168.0.0 0.0.14.0
R2(config)# router ospf 1
R2(config-router)# distribute-list 2 in
R2(config-router)# do write
Building configuration...
[OK]
```

### 📝 Command Explanation

| Command | What it does |
|---------|--------------|
| `access-list 1 permit 192.168.1.0 0.0.14.0` | Permits the network addresses whose third octet is **odd (1–15)** |
| `access-list 2 permit 192.168.0.0 0.0.14.0` | Permits the network addresses whose third octet is **even (0–14)** |
| `router ospf 1` | Enters the OSPF process |
| `distribute-list <acl> in` | Installs only the routes permitted by the ACL into R2's routing table |
| `do write` | Saves the configuration from inside a config mode |

> ⚠️ The ACLs have **no explicit deny** — the **implicit deny at the end** blocks everything not permitted.

---

## 🔍 Verification (Stage by Stage)

### ⚪ Stage 1 — Baseline (expected)

Before any filter, R2 installs **every** external route that R1 redistributes (`O E2 … via 1.1.1.1`). *(Capture `show ip route ospf` here to document the "before".)*

---

### 🟢 Stage 2 — Odd Networks Only (ACL 1)

```
R2(config)# access-list 1 permit 192.168.1.0 0.0.14.0
R2(config)# router ospf 1
R2(config-router)# distribute-list 1 in
*Nov 11 15:42:01.347: %SYS-5-CONFIG_I: Configured from console by console

R2# show ip route
Gateway of last resort is not set

      1.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        1.1.1.0/24 is directly connected, FastEthernet0/0
L        1.1.1.2/32 is directly connected, FastEthernet0/0
O E2  192.168.1.0/24  [110/20] via 1.1.1.1, 00:00:11, FastEthernet0/0
O E2  192.168.3.0/24  [110/20] via 1.1.1.1, 00:00:11, FastEthernet0/0
O E2  192.168.5.0/24  [110/20] via 1.1.1.1, 00:00:11, FastEthernet0/0
O E2  192.168.7.0/24  [110/20] via 1.1.1.1, 00:00:11, FastEthernet0/0
O E2  192.168.9.0/24  [110/20] via 1.1.1.1, 00:00:11, FastEthernet0/0
O E2  192.168.11.0/24 [110/20] via 1.1.1.1, 00:00:11, FastEthernet0/0
O E2  192.168.13.0/24 [110/20] via 1.1.1.1, 00:00:11, FastEthernet0/0
O E2  192.168.15.0/24 [110/20] via 1.1.1.1, 00:00:11, FastEthernet0/0
```

#### 📌 Annotated Reading

| Marker | Explanation |
|--------|-------------|
| `%SYS-5-CONFIG_I` | Configuration applied from the console |
| `O E2` | OSPF **external type 2** (redistributed) routes |
| `[110/20]` | **AD 110 (OSPF)**, **metric 20** (default E2) |
| `via 1.1.1.1` | Next hop = R1 |
| `00:00:11` | Routes were just re-installed after the filter changed |
| **Only `.1 .3 .5 … .15`** | The ACL permitted just the **odd** networks; the even ones were blocked by the implicit deny |
| `C` / `L` | Connected and local routes are **not** affected by the filter |

✅ **Result:** R2 shows only the **8 odd networks**.

---

### 🔵 Stage 3 — Even Networks Only (ACL 2)

```
R2(config)# access-list 2 permit 192.168.0.0 0.0.14.0
R2(config)# router ospf 1
R2(config-router)# distribute-list 2 in
R2(config-router)# do wr
Building configuration...
[OK]

R2# show ip route
      1.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        1.1.1.0/24 is directly connected, FastEthernet0/0
L        1.1.1.2/32 is directly connected, FastEthernet0/0
O E2  192.168.2.0/24  [110/20] via 1.1.1.1, 00:01:33, FastEthernet0/0
O E2  192.168.4.0/24  [110/20] via 1.1.1.1, 00:01:33, FastEthernet0/0
O E2  192.168.6.0/24  [110/20] via 1.1.1.1, 00:01:33, FastEthernet0/0
O E2  192.168.8.0/24  [110/20] via 1.1.1.1, 00:01:33, FastEthernet0/0
O E2  192.168.10.0/24 [110/20] via 1.1.1.1, 00:01:33, FastEthernet0/0
O E2  192.168.12.0/24 [110/20] via 1.1.1.1, 00:01:33, FastEthernet0/0
O E2  192.168.14.0/24 [110/20] via 1.1.1.1, 00:01:33, FastEthernet0/0
```

#### 📌 Annotated Reading

| Marker | Explanation |
|--------|-------------|
| **Only `.2 .4 .6 … .14`** | ACL 2 permits the **even** networks |
| **Odd networks are gone** | The second `distribute-list … in` **replaced** the first one (only one inbound list per OSPF process) |
| Same `[110/20]` / next hop | Only the *selection* changed — the route attributes did not |
| `00:01:33` | Route age after the filter change |
| `.16` missing | `192.168.16.0` matches neither wildcard range (bit 4 is set) |

✅ **Result:** R2 shows only the **7 even networks** (2–14).

---

### 🔄 Stage Summary

| Stage | Config on R2 | Odd (1–15) | Even (2–14) |
|-------|--------------|:----------:|:-----------:|
| 1 | No filter (expected) | ✅ | ✅ |
| 2 | `distribute-list 1 in` | ✅ **only** | ❌ |
| 3 | `distribute-list 2 in` | ❌ | ✅ **only** |

---

## ⚙️ Distribute-List Behavior & Limits

| Fact | Explanation |
|------|-------------|
| **Filters installation, not LSAs** | The OSPF **LSDB on R2 still holds every external LSA**. Only the RIB (routing table) is filtered. |
| **Local effect only** | Other routers (including those behind R2) are **not** affected — they still receive and calculate everything. |
| **One inbound list per process** | A second `distribute-list <acl> in` **replaces** the first. That is why the odd routes disappeared in Stage 3. |
| **Implicit deny** | Anything not matched by a `permit` is blocked. |
| **Standard ACL matches the prefix** | It evaluates the **network address**, not the mask length. |
| **Connected/local routes** | Unaffected. |
| **Not a security feature** | Traffic to a "filtered" prefix is simply not routed by R2; the LSDB still knows about it. |

### 🔀 Combining odd **and** even

Only one inbound list can exist at a time, so to permit both sets use **one ACL with two lines** (or a prefix-list):

```bash
R2(config)# access-list 3 permit 192.168.1.0 0.0.14.0
R2(config)# access-list 3 permit 192.168.0.0 0.0.14.0
R2(config)# router ospf 1
R2(config-router)# distribute-list 3 in
```

---

## 🧰 Other Ways to Filter OSPF

| Method | Where it works | Notes |
|--------|----------------|-------|
| **`distribute-list in`** ⭐ | Local RIB installation | Simple; LSDB unchanged |
| **`distribute-list prefix-list`** | Local RIB installation | Matches **prefix and length** (more precise than ACL) |
| **`distribute-list route-map`** | Local RIB installation | Combine with other match criteria |
| **`area <id> filter-list`** | On an **ABR** | Filters inter-area (Type 3) LSAs between areas |
| **`area <id> range … not-advertise`** | On an **ABR** | Suppresses specific inter-area summaries |
| **`summary-address … not-advertise`** | On an **ASBR** | Suppresses external routes at the source |
| **Redistribution route-map** | On the **ASBR** (R1 here) | Stops unwanted routes from entering OSPF at all |
| **Stub / NSSA areas** | Area design | Blocks external LSAs from an area |

> 💡 In production, filtering **at the source (R1/ASBR)** is cleaner than filtering at every receiver, because it keeps the LSDB small.

---

## 🧾 Command Cheat Sheet

| Purpose | Command |
|---------|---------|
| Standard ACL | `access-list <1-99> permit <net> <wildcard>` |
| Inbound filter | `distribute-list <acl> in` (under `router ospf`) |
| Outbound filter (redistribution) | `distribute-list <acl> out <protocol>` |
| Static route to discard | `ip route <net> <mask> Null0` |
| Redistribute static into OSPF | `redistribute static subnets` |
| Routing table | `show ip route` / `show ip route ospf` |
| OSPF external LSAs | `show ip ospf database external` |
| Confirm the ACL | `show access-lists` |
| Confirm the filter | `show ip protocols` |
| OSPF neighbors | `show ip ospf neighbor` |

---

## 🧠 Key Takeaways

1. 🎛️ A **distribute-list in** controls which OSPF routes are **installed in the routing table** on that router.
2. 🔢 The wildcard **`0.0.14.0`** matches **all odd** (base `.1`) or **all even** (base `.0`) third-octet values from 0–15.
3. 🧮 In the wildcard `14` = `00001110`, bit 0 and bits 4–7 must match; bits 1–3 are free.
4. 🏷️ **`O E2`** routes are **redistributed external routes**, default metric **20**, AD **110** → `[110/20]`.
5. ❌ The ACL has an **implicit deny**, so every non-permitted route is blocked.
6. 🔁 Only **one inbound distribute-list** is allowed per OSPF process — the second one **replaced** the first.
7. 📚 The **LSDB is not filtered**; only the routing table is. Other routers still receive everything.
8. 📍 The filter is **local**: it does not change what R2 floods or what R2's neighbors calculate.
9. 🕳️ `ip route … Null0` + `redistribute static` is a quick way to generate many test routes.
10. 🧱 For production, filter at the **ASBR** (or use prefix-lists/route-maps) for cleaner, more precise control.

---

## 🩺 Troubleshooting Checklist

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| **All** external routes disappeared | ACL doesn't match the real prefixes (implicit deny) | Check the ACL network address and wildcard against `show ip route` |
| Odd routes vanished after configuring the even list | Second `distribute-list in` replaced the first | Use one ACL containing both permits |
| One prefix (e.g., `.16`) missing | Outside the wildcard range | Add a dedicated `permit` line |
| Routes still present after adding the list | OSPF hasn't recalculated | Wait for SPF, or `clear ip ospf process` |
| ACL permits but nothing installs | Wrong direction or wrong process ID | Verify `distribute-list` is under the correct `router ospf` |
| Neighbors behind R2 still see the routes | Distribute-list is local only | Filter at the ASBR or on each router |
| No `O E2` routes at all | R1 not redistributing / adjacency down | `show ip ospf neighbor`; check `redistribute static subnets` |

---

## 💡 Ideas to Extend This Lab

- 🔬 Capture `show ip ospf database external` before and after to **prove the LSDB is unchanged**
- 🔁 Replace the ACL with a **prefix-list** and filter by prefix length
- 🎯 Move the filter to **R1** with a redistribution **route-map** (source-side filtering)
- ➕ Add a **third router** behind R2 and show the filter is **local only**
- 🧪 Permit odd and even **together** with a single ACL
- 📈 Redistribute with `metric-type 1` and compare **E1 vs E2** costs

---

## 🧰 Skills Demonstrated

`OSPF` · `Route Filtering` · `distribute-list` · `Standard ACLs` · `Wildcard Masks` · `Route Redistribution` · `OSPF External Routes (E2)` · `Null0 Static Routes` · `Cisco IOS` · `GNS3` · `Routing Table Analysis` · `Network Troubleshooting` · `Network Documentation`

---

## 👤 Author

<div align="center">

**Maaz Khan**
CCNA-Certified Network & NOC Engineer
📍 Lower Dir, KPK, Pakistan

![CCNA](https://img.shields.io/badge/CCNA-Certified-blue?style=flat-square&logo=cisco)
![NOC](https://img.shields.io/badge/NOC-Engineer-orange?style=flat-square)

⭐ If this lab helped you, give the repo a star!

</div>

---

<div align="center">

**#OSPF #DistributeList #RouteFiltering #ACL #WildcardMask #CCNA #CCNP #Cisco #GNS3 #Networking #NetworkEngineer #NOC**

</div>
