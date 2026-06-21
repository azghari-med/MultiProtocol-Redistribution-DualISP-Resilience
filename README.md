<div align="center">

# 🌐 Multi-Protocol Redistribution \& Dual-ISP Resilience

### "After the Merger" — A Real-World Routing Integration Lab





## 🏢 The Scenario

**OmniMerge Inc.** just acquired a second company. Now the network team faces a classic real-world problem: **two networks that don't speak the same routing language.**

* The **original company** runs **OSPF**
* The **acquired company** runs **EIGRP**
* A few **legacy branch segments** still run **RIP**
* The business wants **two ISPs** for redundancy — and wants **OSPF-side traffic to prefer ISP1**, **EIGRP-side traffic to prefer ISP2**
* If **either ISP fails**, that side must **automatically reroute through the other ISP**

Your job: stitch it all together into one network that routes cleanly by design, and survives an ISP failure — without manual intervention.

> 💡 This is exactly what happens after mergers, acquisitions, and migrations. Redistribution between protocols is a core \\\*\\\*ENARSI\\\*\\\* skill, and dual-ISP resilience is a daily reality for enterprises.

\---

## 🗺️ Topology

```
                          ☁️  INTERNET
                         /            \\\\
                   ISP1 (R1)        ISP2 (R2)
                       \\\\    IP SLA    /
                        \\\\  tracking  /
                      ┌──────────────────┐
                      │  R3 — BORDER /   │
                      │  REDISTRIBUTION  │   ◄── the brain
                      └────────┬─────────┘
                  ┌────────────┴────────────┐
              OSPF│                         │ EIGRP
              ┌───┴────┐               ┌────┴────┐
              │   R4   │               │   R5    │
              └──┬───┬─┘               └──┬───┬──┘
            RIP  │   │ OSPF         EIGRP │   │  RIP
            ┌────┘   └────┐         ┌─────┘   └────┐
          ┌─┴─┐         ┌─┴─┐     ┌─┴─┐           ┌┴─┐
          │R6 │         │R7 │     │R8 │           │R9│
          └─┬─┘         └─┬─┘     └─┬─┘           └─┬┘
         clients       clients   clients         clients
         (RIP)        (OSPF stub)(EIGRP stub)    (RIP)
```

|Router|Role|Routing|
|-|-|-|
|**R1**|ISP1 edge|Static / BGP to internet|
|**R2**|ISP2 edge|Static / BGP to internet|
|**R3**|**Border + Redistribution**|OSPF ↔ EIGRP ↔ RIP, PBR, IP SLA|
|**R4**|OSPF domain head|OSPF (+ RIP edge)|
|**R5**|EIGRP domain head|EIGRP (+ RIP edge)|
|**R6**|OSPF-side branch|RIP|
|**R7**|OSPF-side branch|OSPF **stub**|
|**R8**|EIGRP-side branch|EIGRP **stub**|
|**R9**|EIGRP-side branch|RIP|

\---

## 🎯 What This Lab Demonstrates

```
✅ Mutual redistribution  — OSPF ↔ EIGRP at the border (R3)
✅ Local redistribution   — RIP ↔ OSPF (R4) and RIP ↔ EIGRP (R5)
✅ Single-point design    — no loops by topology (one boundary each)
✅ Stub areas             — OSPF stub \\\& EIGRP stub
✅ Path preference        — OSPF→ISP1, EIGRP→ISP2 (PBR)
✅ Dual-ISP failover      — IP SLA + object tracking
✅ NAT / PAT              — internal 10.x → public ISP IP (dual-ISP)
✅ Route control          — prefix-lists \\\& route-maps
```

\---

## 🧠 The Design Challenges (and How They're Solved)

### Design Note — Why There Are No Redistribution Loops Here

> Mutual redistribution becomes dangerous when \\\*\\\*two or more routers\\\*\\\* bridge the same pair of domains — a route can leave one protocol, re-enter through the second boundary, and loop.

**This topology avoids that by design.** Every redistribution boundary is a **single router**:

```
R3  → the ONLY link between OSPF and EIGRP
R4  → the ONLY redistribution point for RIP (R6) ↔ OSPF
R5  → the ONLY redistribution point for RIP (R9) ↔ EIGRP
```

The two RIP segments (R6 and R9) live on **opposite sides** and never connect to each other, so no route ever has a second path back into its origin. **No loops are possible — so no route tags are needed.**

> 🎓 \\\*\\\*The professional point:\\\*\\\* route tagging \\\*would\\\* be required if there were multiple redistribution points between the same domains. Knowing \\\*when\\\* you need it — and when you don't — is the difference between configuring blindly and designing deliberately.

### Per-Domain ISP Preference

> The business wants OSPF traffic on ISP1 and EIGRP traffic on ISP2 — but both share the same border router.

**Solution:** **Policy-Based Routing (PBR)**. A route-map matches the source domain and sets the next hop to the correct ISP.

### ISP Failure with No Manual Fix

> If ISP1 dies, OSPF traffic must move to ISP2 automatically — and back when ISP1 recovers.

**Solution:** **IP SLA** continuously pings each ISP. A **track object** watches the SLA. PBR uses `verify-availability` so traffic only takes an ISP that's actually reachable — failing over instantly.

\---

## ⚙️ Build Phases

|Phase|Focus|
|:-:|-|
|**1**|Base connectivity \& interfaces|
|**2**|OSPF domain (with stub area)|
|**3**|EIGRP domain (with stub)|
|**4**|RIP legacy segments|
|**5**|Redistribution (R3: OSPF↔EIGRP · R4/R5: RIP)|
|**6**|Dual-ISP path preference (PBR)|
|**7**|IP SLA failover|
|**8**|NAT — internal 10.x → public ISP IP|

\---

## 🔵 PHASE 2 — OSPF Domain

```cisco
! R4 — OSPF domain head
router ospf 1
 router-id 4.4.4.4
 network 10.4.0.0 0.0.255.255 area 0
 network 10.47.0.0 0.0.0.255 area 10

! Area 10 as STUB (R7 is a stub branch)
 area 10 stub

! R7 — OSPF stub branch
router ospf 1

&#x20;router-id 7.7.7.7

&#x20;area 10 stub

&#x20;passive-interface default

&#x20;no passive-interface Ethernet0/0

&#x20;network 10.47.0.0 0.0.255.255 area 10

&#x20;network 10.70.0.0 0.0.255.255 area 10

```

> \\\*\\\*Why stub:\\\*\\\* R7's branch doesn't need full external routes — a default route is enough. Stub areas shrink the routing table and protect the branch from external route churn.

\---

## 🟢 PHASE 3 — EIGRP Domain

```cisco
! R5 — EIGRP domain head
router eigrp 100
 network 10.5.0.0 0.0.255.255
 network 10.58.0.0 0.0.0.255
 no auto-summary

! R8 — EIGRP STUB branch
router eigrp 100
 network 10.58.0.0 0.0.0.255
 eigrp stub connected summary
 no auto-summary
```

> \\\*\\\*Why EIGRP stub:\\\*\\\* R8 only originates its own connected routes — it will never be used as a transit path. This reduces query scope and speeds convergence.

\---

## 🟡 PHASE 4 — RIP Legacy Segments

```cisco
! R6 and R9 — legacy RIP branches
router rip
 version 2
 network 10.6.0.0
 no auto-summary
```

\---

## 🔴 PHASE 5 — Redistribution (THE CORE)

Redistribution happens at **three single-point boundaries** — each handled by exactly one router, so no loops can form.

### Task 5.1 — R3: Mutual OSPF ↔ EIGRP

```cisco
! On R3 — the ONLY link between the OSPF and EIGRP domains

! EIGRP needs a seed metric; OSPF needs the subnets keyword
router eigrp 100
 redistribute ospf 1 metric 10000 100 255 1 1500

router ospf 1
 redistribute eigrp 100 subnets
```

### Task 5.2 — R4: RIP ↔ OSPF (OSPF side only)

```cisco
! On R4 — the ONLY redistribution point for R6's RIP segment
router ospf 1
 redistribute rip subnets

router rip
 redistribute ospf 1 metric 5
```

### Task 5.3 — R5: RIP ↔ EIGRP (EIGRP side only)

```cisco
! On R5 — the ONLY redistribution point for R9's RIP segment
router eigrp 100
 redistribute rip metric 10000 100 255 1 1500

router rip
 redistribute eigrp 100 metric 5
```

> \\\*\\\*Why no route tags?\\\*\\\* Each boundary has a single redistribution router, and the two RIP segments (R6, R9) sit on opposite sides and never connect. No route has a second path back into its origin, so loops are impossible by topology. Tags would only be required with \\\*\\\*multiple\\\*\\\* redistribution points between the same domains.

> ⚠️ \\\*\\\*ASBR + stub note:\\\*\\\* R4 and R5 redistribute RIP, which makes them \\\*\\\*ASBRs\\\*\\\*. An ASBR \\\*\\\*cannot\\\*\\\* sit in a pure stub area — so R4 and R5 live in \\\*\\\*area 0 (backbone)\\\*\\\*, while the stub branch (R7, area 10) stays separate, with R4 acting as the ABR between them.

\---

## 🌍 PHASE 6 — Dual-ISP Path Preference (PBR)

R3 has **two internal interfaces** — `e0/3` facing the OSPF domain (R4), `e0/2` facing the EIGRP domain (R5). PBR acts on **ingress** traffic, so:

* Traffic from the **OSPF** domain can only enter via **e0/3** → push it to **ISP1**
* Traffic from the **EIGRP** domain can only enter via **e0/2** → push it to **ISP2**

Because the **interface itself identifies the domain**, no source-IP matching is needed — each interface gets its own simple route-map.

```cisco
! Match ONLY internet-bound traffic — exclude internal-to-internal
! (so cross-domain internal traffic is routed normally, NOT sent to an ISP)
ip access-list extended PBR-OSPF-ACL
 deny   ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255   ! internal = skip PBR
 permit ip 10.0.0.0 0.255.255.255 any                       ! internet = apply PBR

ip access-list extended PBR-EIGRP-ACL
 deny   ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255
 permit ip 10.0.0.0 0.255.255.255 any

! ---- OSPF traffic enters from R4 (e0/3) → prefer ISP1 ----
route-map PBR-OSPF permit 10
 match ip address PBR-OSPF-ACL
 set ip next-hop 203.0.113.1        ! ISP1 (R1)

interface Ethernet0/3                ! R3 → R4 (OSPF domain)
 ip policy route-map PBR-OSPF

! ---- EIGRP traffic enters from R5 (e0/2) → prefer ISP2 ----
route-map PBR-EIGRP permit 10
 match ip address PBR-EIGRP-ACL
 set ip next-hop 198.51.100.1       ! ISP2 (R2)

interface Ethernet0/2                ! R3 → R5 (EIGRP domain)
 ip policy route-map PBR-EIGRP
```

> ⚠️ \\\*\\\*The internal-exclusion ACL is essential.\\\*\\\* Without the `deny ip 10.x 10.x` line, PBR grabs internal cross-domain traffic (e.g. R6→R8) and forces it out to an ISP — breaking internal connectivity even though the routes exist. Match only internet-bound traffic.

> 💡 \\\*\\\*Why match an ACL at all:\\\*\\\* the ingress interface already identifies the domain, but you still need the ACL to separate \\\*internet-bound\\\* traffic (apply PBR) from \\\*internal\\\* traffic (route normally). The interface picks the ISP; the ACL decides whether PBR applies at all.

\---

## ⚡ PHASE 7 — IP SLA Failover (THE STAR FEATURE)

### Task 7.1 — Monitor Each ISP

```cisco
! Ping each ISP continuously (ISP1 via e0/1, ISP2 via e0/0)
ip sla 1
 icmp-echo 203.0.113.1 source-interface Ethernet0/1
 frequency 5
ip sla schedule 1 life forever start-time now

ip sla 2
 icmp-echo 198.51.100.1 source-interface Ethernet0/0
 frequency 5
ip sla schedule 2 life forever start-time now

track 1 ip sla 1 reachability     ! ISP1 health
track 2 ip sla 2 reachability     ! ISP2 health
```

### Task 7.2 — Add Failover to Each Route-Map

```cisco
! OSPF interface: ISP1 first, fail over to ISP2
route-map PBR-OSPF permit 10
 match ip address PBR-OSPF-ACL
 set ip next-hop verify-availability 203.0.113.1 1 track 1   ! ISP1 first
 set ip next-hop verify-availability 198.51.100.1 2 track 2  ! ISP2 backup

! EIGRP interface: ISP2 first, fail over to ISP1
route-map PBR-EIGRP permit 10
 match ip address PBR-EIGRP-ACL
 set ip next-hop verify-availability 198.51.100.1 1 track 2  ! ISP2 first
 set ip next-hop verify-availability 203.0.113.1 2 track 1   ! ISP1 backup
```

> \\\*\\\*How it works:\\\*\\\* Each interface's route-map prefers its assigned ISP. If that ISP's IP SLA fails, the track object drops, `verify-availability` skips the dead next hop, and traffic uses the other ISP. When the ISP recovers, traffic returns. \\\*\\\*Fully automatic — no manual intervention.\\\*\\\*

> ⚠️ \\\*\\\*Don't leave a plain `set ip next-hop` line alongside the `verify-availability` lines\\\*\\\* — a leftover plain next-hop overrides the tracked ones and forces ALL traffic to one ISP regardless of failover. Keep only the `verify-availability` lines.

\---

## 🟤 PHASE 8 — NAT (Reach the Real Internet)

The internal networks use private `10.x` addressing, which **cannot be routed across the real internet**. R3 must translate internal sources to its public ISP-facing IP.

### Task 8.1 — Mark Inside / Outside Interfaces

```cisco
! Internal (facing R4/R5) = inside
interface Ethernet0/2
 ip nat inside
interface Ethernet0/3
 ip nat inside

! ISP-facing = outside
interface Ethernet0/0
 ip nat outside
interface Ethernet0/1
 ip nat outside
```



### Task 8.2 — Translate Internal 10.x for BOTH ISPs (exclude internal-to-internal)

```cisco
! CRITICAL: only NAT internet-bound traffic.
! The deny line stops internal cross-domain traffic (10.x→10.x)
! from being translated toward the ISP.
ip access-list extended NAT-NETS
 deny   ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255   ! 10.x→10.x = NO NAT
 permit ip 10.0.0.0 0.255.255.255 any                       ! 10.x→internet = NAT

! One route-map per ISP exit — so traffic out EITHER ISP is translated
route-map NAT-ISP1 permit 10
 match ip address NAT-NETS
 match interface Ethernet0/1
route-map NAT-ISP2 permit 10
 match ip address NAT-NETS
 match interface Ethernet0/0

ip nat inside source route-map NAT-ISP1 interface Ethernet0/1 overload
ip nat inside source route-map NAT-ISP2 interface Ethernet0/0 overload
```

> ⚠️ \\\*\\\*Without the `deny` line\\\*\\\*, internal cross-domain traffic (e.g. R6 10.60.x → R8 10.80.x) gets NATted toward the ISP and breaks — even though the routes exist. Always exclude internal-to-internal. And after any NAT ACL change, run `clear ip nat translation \\\*`.

> \\\*\\\*Why two route-maps:\\\*\\\* OSPF traffic exits ISP1 (e0/1), EIGRP traffic exits ISP2 (e0/0). A single NAT statement bound to one interface would only translate traffic leaving that one ISP. A route-map per ISP ensures \\\*\\\*both\\\*\\\* exit paths get translated to the correct public IP.

### Task 8.3 — Verify

```cisco
R4# ping 8.8.8.8                  → reaches the internet ✅
R3# show ip nat translations      → see 10.x → public mappings ✅
R3# show ip nat statistics        → hit counts increasing ✅
```

\---

## 🧩 Complete R3-BORDER Configuration

R3 is the heart of the lab — it sits between both ISPs and both routing domains. Here's its full config in one place (interfaces match the PNetLab topology).

```cisco
hostname R3-BORDER

! ===== INTERFACES =====
! Internal interfaces = NAT INSIDE, ISP interfaces = NAT OUTSIDE
interface Ethernet0/0
 description To ISP2 (R2)
 ip address 198.51.100.3 255.255.255.0
 ip nat outside
 no shutdown

interface Ethernet0/1
 description To ISP1 (R1)
 ip address 203.0.113.3 255.255.255.0
 ip nat outside
 no shutdown

interface Ethernet0/2
 description To R5 (EIGRP domain)
 ip address 10.5.0.3 255.255.255.0
 ip nat inside
 ip policy route-map PBR-EIGRP
 no shutdown

interface Ethernet0/3
 description To R4 (OSPF domain)
 ip address 10.4.0.3 255.255.255.0
 ip nat inside
 ip policy route-map PBR-OSPF
 no shutdown

! ===== ROUTING PROTOCOLS =====
router ospf 1
 router-id 3.3.3.3
 network 10.4.0.0 0.0.0.255 area 0
 redistribute eigrp 100 subnets
 default-information originate

router eigrp 100
 network 10.5.0.0 0.0.0.255
 redistribute ospf 1 metric 10000 100 255 1 1500
 redistribute static
 no auto-summary

! ===== DEFAULT ROUTES TO ISPs =====
ip route 0.0.0.0 0.0.0.0 203.0.113.1 10      ! via ISP1 (primary-ish)
ip route 0.0.0.0 0.0.0.0 198.51.100.1 20     ! via ISP2 (backup)

! ===== NAT — translate internal 10.x to the public ISP IP =====
! Extended ACL: exclude internal-to-internal, NAT only internet-bound
ip access-list extended NAT-NETS
 deny   ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255
 permit ip 10.0.0.0 0.255.255.255 any

! One NAT route-map per ISP exit (so BOTH ISPs translate correctly)
route-map NAT-ISP1 permit 10
 match ip address NAT-NETS
 match interface Ethernet0/1
route-map NAT-ISP2 permit 10
 match ip address NAT-NETS
 match interface Ethernet0/0

ip nat inside source route-map NAT-ISP1 interface Ethernet0/1 overload
ip nat inside source route-map NAT-ISP2 interface Ethernet0/0 overload

! ===== IP SLA (ISP monitoring) =====
ip sla 1
 icmp-echo 203.0.113.1 source-interface Ethernet0/1
 frequency 5
ip sla schedule 1 life forever start-time now

ip sla 2
 icmp-echo 198.51.100.1 source-interface Ethernet0/0
 frequency 5
ip sla schedule 2 life forever start-time now

track 1 ip sla 1 reachability
track 2 ip sla 2 reachability

! ===== PBR — PER-DOMAIN ISP PREFERENCE + FAILOVER =====
! Internal-exclusion ACLs: only policy-route internet-bound traffic
ip access-list extended PBR-OSPF-ACL
 deny   ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255
 permit ip 10.0.0.0 0.255.255.255 any
ip access-list extended PBR-EIGRP-ACL
 deny   ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255
 permit ip 10.0.0.0 0.255.255.255 any

route-map PBR-OSPF permit 10
 match ip address PBR-OSPF-ACL
 set ip next-hop verify-availability 203.0.113.1 1 track 1
 set ip next-hop verify-availability 198.51.100.1 2 track 2

route-map PBR-EIGRP permit 10
 match ip address PBR-EIGRP-ACL
 set ip next-hop verify-availability 198.51.100.1 1 track 2
 set ip next-hop verify-availability 203.0.113.1 2 track 1
```

> \\\*\\\*Reading R3's interfaces:\\\*\\\* `e0/1`→ISP1, `e0/0`→ISP2, `e0/3`→R4 (OSPF), `e0/2`→R5 (EIGRP).
> - \\\*\\\*NAT direction:\\\*\\\* internal interfaces (e0/2, e0/3) = `ip nat inside`; ISP interfaces (e0/0, e0/1) = `ip nat outside`.
> - \\\*\\\*PBR\\\*\\\* is applied \\\*\\\*inbound\\\*\\\* on the two internal interfaces — the interface a packet arrives on decides which ISP it prefers.
> - \\\*\\\*Internal exclusion:\\\*\\\* both NAT and PBR ACLs `deny 10.x→10.x` so internal cross-domain traffic routes normally and is never translated or redirected to an ISP.
> - \\\*\\\*No leftover plain `set ip next-hop`\\\*\\\* — only `verify-availability` lines, so failover actually works.

\---

## 🧪 EIGRP Stub Verification — Transit Test (R10)

To prove the EIGRP stub on **R8** is not used as a transit path, add a router **R10** with 4 loopbacks in a **triangle** with R5 and R8.

```
         R5 ───────────────── R8 (EIGRP stub)
          \\\\                  /
   10.105.0.0/24      10.108.0.0/24
            \\\\              /
             R10 (Lo1-4: 172.16.1.1-4)
```

R10's loopbacks are reachable two ways — directly via R5, or through R8. With R8 as stub, R8 refuses to advertise R10's routes, so R5 uses only the direct path.

### Test A — R8 STUB: only ONE path

```cisco
R5# show ip eigrp topology all-links 172.16.1.1/32
→ ONE path: via 10.105.0.10 (R10 direct) ✅
```

### Test B — R8 NON-STUB: TWO paths appear

```cisco
R8(config-router)# no eigrp stub
R5# show ip eigrp topology all-links 172.16.1.1/32
→ via 10.105.0.10 (409600 — direct, successor) ✅
→ via 10.58.0.8   (435200 — through R8, backup) ✅
```

> Use `show ip eigrp topology all-links` — the R8 path is a higher-metric backup the standard view hides. Direct wins on metric; R8 is the backup.

### Test C — The Proof: break the direct link

|R8 state|After shutting the R5→R10 direct link|
|-|-|
|**Non-stub**|Route survives **via R8** — R8 acts as transit ✅|
|**Stub**|Route is **GONE** — R8 refuses transit ❌|

> \\\*\\\*Why:\\\*\\\* a stub router advertises only its own connected/summary routes, never routes learned from another neighbor (R10). That blocks transit and shrinks EIGRP query scope.

\---

## 🔬 The Failover Test (Money Shot)

```cisco
Step 1 — Confirm normal state
  R3# show track brief
  → Track 1 (ISP1): up    Track 2 (ISP2): up ✅
  → OSPF traffic via ISP1, EIGRP traffic via ISP2 ✅

Step 2 — Continuous ping from an OSPF-side client to the internet
  C> ping 8.8.8.8 -t          (going out ISP1)

Step 3 — Kill ISP1
  R1(config)# interface \\\[internet-facing]
  R1(config-if)# shutdown

Step 4 — IP SLA detects, track drops
  R3# show track brief
  → Track 1 (ISP1): DOWN ❌

Step 5 — OSPF traffic reroutes to ISP2 automatically ✅
  Ping continues (a few drops, then recovers) ✅

Step 6 — Restore ISP1
  R1(config-if)# no shutdown
  → Track 1 recovers → OSPF traffic returns to ISP1 ✅
```

\---

## ✅ Verification

```cisco
Redistribution:
  show ip route ospf    → EIGRP \\\& RIP routes as O E2 ✅
  show ip route eigrp   → OSPF \\\& RIP routes as D EX ✅
  show ip route rip     → legacy nets reachable ✅

Single best paths:
  show ip route 10.4.0.0   → single best path, no flapping ✅

Stub areas:
  show ip ospf            → area 10 stub ✅
  show ip eigrp neighbors → R8 stub ✅

Path preference:
  traceroute from OSPF client → exits ISP1 ✅
  traceroute from EIGRP client → exits ISP2 ✅

IP SLA failover:
  show track brief        → both up ✅
  show ip sla statistics  → returns OK ✅
```

\---

## 🐛 Troubleshooting Highlights

### 🔴 REAL BUG — Internal Cross-Domain Traffic Fails (Routes Exist!)

> \\\*\\\*Symptom:\\\*\\\* R6 (OSPF side) and R8 (EIGRP side) see each other's networks in the routing table, but can't ping across. Internet works — only internal cross-domain fails.

```
Diagnosis: routing is NOT the problem
  R6, R8, and R3 all HAVE the routes ✅
  Yet ping fails ❌

Root cause: R3 was hijacking internal traffic
  NAT translated 10.60→10.80 toward the ISP ❌
  PBR policy-routed internal traffic to an ISP ❌
  (plus a leftover plain set ip next-hop forcing one ISP)

Fix: exclude internal-to-internal from BOTH NAT and PBR
  deny ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255
  permit ip 10.0.0.0 0.255.255.255 any
  + remove leftover plain "set ip next-hop"
  + clear ip nat translation \\\*
```

**Lesson:** the routing table can look perfect while a data-plane policy (NAT or PBR) silently grabs the packet. Any redirect/translate policy must exclude internal-to-internal traffic.

### Quick Reference

|Problem|Cause|Fix|
|-|-|-|
|Routes missing after redistribution|Missing seed metric / subnets keyword|Add `metric` (EIGRP) or `subnets` (OSPF)|
|Suboptimal path|Redistributed metric too attractive|Tune seed metrics / admin distance|
|PBR not taking effect|route-map not applied to interface|`ip policy route-map` on ingress interface|
|Failover doesn't trigger|track not referenced in PBR|use `verify-availability ... track N`|
|Internal cross-domain ping fails|NAT/PBR catching internal traffic|`deny 10.x→10.x` in both ACLs|
|All traffic forced to one ISP|leftover plain `set ip next-hop`|remove it, keep only `verify-availability`|
|Internal hosts can't reach internet|NAT inside/outside swapped|inside = R4/R5 side, outside = ISP side|
|Only one ISP's traffic translates|NAT bound to a single interface|use a NAT route-map per ISP exit|
|PBR drops traffic|`verify-availability track` with no track defined|create IP SLA + track first|
|NAT ACL changed, behavior persists|stale translations|`clear ip nat translation \\\*`|
|Stub branch missing routes|stub too restrictive|use correct stub type (e.g. `eigrp stub connected summary`)|

\---

## 📚 Key Concepts

<details>
<summary><b>Why this design has no loops (and when you'd need tags)</b></summary>

```
A redistribution loop needs TWO+ points bridging
the same domains. This topology has ONE router per
boundary:
  R3 → OSPF↔EIGRP
  R4 → RIP↔OSPF
  R5 → RIP↔EIGRP

R6 and R9 (RIP) sit on opposite sides and never meet.
So no route can re-enter its origin by another path.
→ No loops. No tags needed.

You WOULD need route tags if two routers redistributed
between the same pair of domains — tag by origin, then
deny tagged routes from re-entering their source. ✅
```

</details>

<details>
<summary><b>Why PBR instead of routing protocols for ISP choice</b></summary>

```
Normal routing picks ONE best path for a destination.
We want path choice based on the SOURCE domain instead.
PBR (policy-based routing) matches source and sets the
next hop — overriding the routing table by policy. ✅
```

</details>

<details>
<summary><b>Why IP SLA + verify-availability</b></summary>

```
PBR alone sends traffic to a next hop even if it's dead.
IP SLA pings the ISP; a track object reflects its health;
verify-availability only uses a next hop whose track is up.
→ instant, automatic failover. ✅
```

</details>

\---

<div align="center">

## 🏆 Why This Lab Stands Out

|Skill|Demonstrated|
|-|-|
|**Redistribution**|OSPF ↔ EIGRP ↔ RIP, single-point boundary design|
|**Route Control**|Seed metrics, prefix-lists, route-maps, admin distance|
|**Scalability**|OSPF \& EIGRP stub design|
|**Policy Routing**|Per-domain ISP preference via PBR|
|**Resilience**|IP SLA + object tracking dual-ISP failover|
|**NAT**|Dual-ISP PAT with route-maps, inside/outside design|

**Redistribution mastery + dual-ISP resilience — core ENARSI, real-world ready.**

</div>

