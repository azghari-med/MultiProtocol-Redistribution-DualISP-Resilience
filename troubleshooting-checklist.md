# 🔧 Troubleshooting Checklist — Multi-Protocol Redistribution & Dual ISP

## ⚡ Diagnostic Order (Build in Phases)

```
Phase 1: Base connectivity (interfaces, IPs, pings)
Phase 2: OSPF domain + stub
Phase 3: EIGRP domain + stub
Phase 4: RIP legacy segments
Phase 5: Redistribution (R3: OSPF↔EIGRP, R4/R5: RIP)
Phase 6: Dual-ISP PBR
Phase 7: IP SLA failover
Phase 8: NAT (internal 10.x → public ISP IP)
```

> Verify each protocol works ALONE before redistributing.

---

## Phase 2-4 — Each Protocol Works Independently

```
OSPF:
  show ip ospf neighbor    → adjacencies FULL ✅
  show ip route ospf       → intra/inter-area routes ✅

EIGRP:
  show ip eigrp neighbors  → neighbors present ✅
  show ip route eigrp      → D routes ✅

RIP:
  show ip route rip        → R routes ✅
  show ip protocols        → RIP v2, no auto-summary ✅
```

If a protocol is broken:
  [ ] Interfaces up, correct IPs
  [ ] network statements match interfaces
  [ ] no auto-summary (EIGRP/RIP)
  [ ] OSPF area IDs match on the link
  [ ] EIGRP AS number matches

---

## Phase 5 — Redistribution (Most Common Problems Here)

```
show ip route             → external routes appear (O E2, D EX, R) ✅
show ip route 10.x.x.x    → SINGLE best path, not flapping ✅
```

### 🟢 No Loops By Design (but verify single best paths)
```
This topology has ONE redistribution router per boundary
(R3, R4, R5) and separated RIP segments — so loops can't form.
Still verify clean paths:
  [ ] show ip route 10.x.x.x → single best path, not flapping
  [ ] If flapping appears, check you didn't accidentally create
      a second redistribution point between the same domains
```

### Routes Missing After Redistribution
```
Cause : seed metric missing (EIGRP needs metric), subnets keyword (OSPF)
Fix   :
  [ ] EIGRP redistribute ... metric 10000 100 255 1 1500
  [ ] OSPF redistribute ... subnets
  [ ] RIP redistribute ... metric N
```

### Suboptimal Routing
```
Cause : redistributed route more attractive than native
Fix   :
  [ ] Adjust seed metric
  [ ] Tune administrative distance if needed
  [ ] Verify only one redistribution point per boundary
```

---

## Phase 6 — PBR Path Preference

```
show route-map PBR-OSPF           → policy + match counts ✅
show route-map PBR-EIGRP          → policy + match counts ✅
show ip policy                    → applied to interface ✅
traceroute (from OSPF client)     → exits via ISP1 ✅
traceroute (from EIGRP client)    → exits via ISP2 ✅
```

If PBR not working:
  [ ] route-map applied with `ip policy route-map` on INGRESS interface
  [ ] ACLs match the correct SOURCE networks
  [ ] next-hop reachable (without track first, to test)

---

## Phase 7 — IP SLA Failover

```
show ip sla statistics            → "Return code: OK" ✅
show track brief                  → Track 1 & 2 up ✅
show ip sla configuration         → correct target/frequency ✅
```

If failover doesn't work:
  [ ] IP SLA scheduled (life forever start-time now)
  [ ] track references the correct sla number
  [ ] PBR uses `set ip next-hop verify-availability ... track N`
  [ ] Test: shut the ISP link → track goes DOWN → traffic reroutes

---

## Phase 8 — NAT (Reach the Internet)

```
show ip nat translations          → 10.x → public IP mappings ✅
show ip nat statistics            → hit counts increasing ✅
```

If internal hosts can't reach the internet:
  [ ] inside/outside on the RIGHT interfaces
      (inside = R4/R5 side, outside = ISP side) ← #1 mistake
  [ ] NAT route-map exists for BOTH ISP exits
  [ ] NAT-NETS ACL permits 10.0.0.0/8
  [ ] R3 itself can ping internet (confirms ISP path)
  [ ] Remember: private 10.x can't cross the real internet —
      NAT is REQUIRED, not optional

### 🔴 Classic NAT Symptom
```
Symptom: R3 pings internet ✅ but R4/clients can't ❌
Cause  : R3 sources from its public IP (works);
         R4 sources from 10.x (ISP can't reply / not NATted)
Fix    : NAT on R3 with correct inside/outside + dual-ISP route-maps
```

---

## 🔴 REAL BUG SOLVED — Internal Cross-Domain Traffic Fails

```
Symptom:
  R6 (OSPF side) and R8 (EIGRP side) SEE each other's
  networks in the routing table, but CANNOT ping across.
  Internet works fine — only internal cross-domain fails.

Diagnosis (routing is NOT the problem):
  R6# show ip route 10.80.0.0   → present ✅
  R8# show ip route 10.60.0.0   → present ✅
  R3# show ip route 10.60.0.0   → via R4 (e0/3) ✅
  R3# show ip route 10.80.0.0   → via R5 (e0/2) ✅
  Routes exist everywhere — yet ping fails ❌

Root cause (R3 hijacking internal traffic):
  1 — NAT ACL matched ALL 10.x → it translated
      internal 10.60→10.80 toward the ISP ❌
  2 — PBR had no exclusion (+ a leftover plain
      set ip next-hop) → it policy-routed internal
      traffic out to the ISP ❌

Both policies grabbed internal-to-internal traffic
that should have routed normally between the sites.
```

### The Fix — Exclude internal-to-internal from BOTH NAT and PBR

```cisco
! NAT — only translate internet-bound traffic
ip access-list extended NAT-NETS
 deny   ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255   ! 10.x→10.x = NO NAT
 permit ip 10.0.0.0 0.255.255.255 any                       ! 10.x→internet = NAT

! PBR — only redirect internet-bound traffic
ip access-list extended PBR-OSPF-ACL
 deny   ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255
 permit ip 10.0.0.0 0.255.255.255 any

route-map PBR-OSPF permit 10
 match ip address PBR-OSPF-ACL
 set ip next-hop verify-availability 203.0.113.1 1 track 1
 set ip next-hop verify-availability 198.51.100.1 2 track 2
 ! remove any leftover plain "set ip next-hop" line!

! Same pattern for PBR-EIGRP-ACL / PBR-EIGRP (next-hops reversed)

! ALWAYS clear translations after changing a NAT ACL
clear ip nat translation *
```

### Verify the Fix
```
R6# ping 10.80.0.8 source 10.60.0.6   → internal works ✅
R6# ping 8.8.8.8                       → internet still works ✅
```

> **Lesson:** any policy that redirects or translates (NAT **and** PBR)
> must EXCLUDE internal-to-internal traffic — match only internet-bound
> with `deny ip <internal> <internal>` then `permit ip <internal> any`.
> The routing table can look perfect while a data-plane policy silently
> hijacks the packet. And after any NAT ACL change → `clear ip nat translation *`.

---

## 🔬 THE FAILOVER TEST

```
Step 1 — show track brief → both up ✅
Step 2 — Continuous ping from OSPF client → 8.8.8.8 (via ISP1)
Step 3 — Shut ISP1 internet link
Step 4 — show track brief → Track 1 DOWN ❌
Step 5 — Ping continues (reroutes to ISP2) ✅
Step 6 — Restore ISP1 → track recovers → traffic returns ✅
```

---

## 🔑 Most Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Second redistribution point added | Routing loop / flapping | Keep one router per boundary (or add tags) |
| NAT inside/outside swapped | Internal hosts no internet | inside = R4/R5 side, outside = ISP side |
| NAT only on one interface | One ISP's traffic not translated | NAT route-map per ISP exit |
| NAT ACL matches all 10.x | Internal cross-domain ping fails | deny 10.x→10.x, permit 10.x→any |
| PBR has no internal exclusion | Internal traffic forced to ISP | deny 10.x→10.x in the PBR ACL |
| Leftover plain `set ip next-hop` | All traffic forced to one ISP | remove it, keep only verify-availability |
| Changed NAT ACL, not cleared | Old behavior persists | `clear ip nat translation *` |
| Missing seed metric (EIGRP) | Routes don't redistribute | Add `metric` keyword |
| Missing `subnets` (OSPF) | Only classful routes appear | Add `subnets` keyword |
| PBR on wrong interface | Policy never matches | Apply on ingress interface |
| No `verify-availability` | Failover doesn't trigger | Add `track N` to next-hop |
| Stub too restrictive | Branch missing routes | Correct stub type |
| `auto-summary` left on | Discontiguous network issues | `no auto-summary` |

---

## ⚡ Key Commands

```
! Routing tables
show ip route
show ip route ospf / eigrp / rip
show ip protocols

! Redistribution / loops
show route-map
show ip ospf database external
show ip eigrp topology

! PBR
show route-map PBR-OSPF
show route-map PBR-EIGRP
show ip policy

! NAT
show ip nat translations
show ip nat statistics
clear ip nat translation *
debug ip nat               (watch what gets translated)

! EIGRP stub transit test
show ip eigrp neighbors detail        (R8 = "Stub Peer")
show ip eigrp topology all-links      (shows backup paths too)

! IP SLA / tracking
show ip sla statistics
show ip sla configuration
show track brief

! Failover proof
debug ip policy           (use carefully)
traceroute <dest>
```
