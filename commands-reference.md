# 📋 Commands Reference — Multi-Protocol Redistribution & Dual ISP

## 🗺️ The Design

```
INTERNET → ISP1(R1) + ISP2(R2) → R3 (border/redistribution)
R3 → OSPF domain (R4 → R6 RIP, R7 OSPF-stub)
R3 → EIGRP domain (R5 → R8 EIGRP-stub, R9 RIP)

Policy: OSPF traffic → ISP1, EIGRP traffic → ISP2
Failover: IP SLA reroutes to the other ISP if one dies
```

---

## 🔵 OSPF Domain

```cisco
! R4 — domain head
router ospf 1
 router-id 4.4.4.4
 network 10.4.0.0 0.0.255.255 area 0
 network 10.47.0.0 0.0.0.255 area 10
 area 10 stub

! R7 — stub branch
router ospf 1
 router-id 7.7.7.7
 network 10.47.0.0 0.0.0.255 area 10
 area 10 stub
```

---

## 🟢 EIGRP Domain

```cisco
! R5 — domain head
router eigrp 100
 network 10.5.0.0 0.0.255.255
 network 10.58.0.0 0.0.0.255
 no auto-summary

! R8 — stub branch
router eigrp 100
 network 10.58.0.0 0.0.0.255
 eigrp stub connected summary
 no auto-summary
```

---

## 🟡 RIP Legacy

```cisco
! R6, R9
router rip
 version 2
 network 10.6.0.0
 no auto-summary
```

---

## 🔴 Redistribution (Single-Point — No Tags Needed)

```cisco
! R3 — the ONLY OSPF↔EIGRP boundary
router eigrp 100
 redistribute ospf 1 metric 10000 100 255 1 1500
router ospf 1
 redistribute eigrp 100 subnets

! R4 — the ONLY RIP↔OSPF boundary (OSPF side)
router ospf 1
 redistribute rip subnets
router rip
 redistribute ospf 1 metric 5

! R5 — the ONLY RIP↔EIGRP boundary (EIGRP side)
router eigrp 100
 redistribute rip metric 10000 100 255 1 1500
router rip
 redistribute eigrp 100 metric 5
```

> No route tags: each boundary is a single router and the two RIP
> segments never connect, so loops are impossible by topology.
> R4 and R5 are ASBRs → they sit in area 0, not a stub area.

---

## 🌍 Dual-ISP PBR (with internal exclusion)

```cisco
! Internal-exclusion ACLs — only policy-route internet-bound traffic
ip access-list extended PBR-OSPF-ACL
 deny   ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255
 permit ip 10.0.0.0 0.255.255.255 any
ip access-list extended PBR-EIGRP-ACL
 deny   ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255
 permit ip 10.0.0.0 0.255.255.255 any

! OSPF traffic enters R3 on e0/3 → prefer ISP1
route-map PBR-OSPF permit 10
 match ip address PBR-OSPF-ACL
 set ip next-hop verify-availability 203.0.113.1 1 track 1
 set ip next-hop verify-availability 198.51.100.1 2 track 2

interface Ethernet0/3          ! R3 → R4 (OSPF)
 ip policy route-map PBR-OSPF

! EIGRP traffic enters R3 on e0/2 → prefer ISP2
route-map PBR-EIGRP permit 10
 match ip address PBR-EIGRP-ACL
 set ip next-hop verify-availability 198.51.100.1 1 track 2
 set ip next-hop verify-availability 203.0.113.1 2 track 1

interface Ethernet0/2          ! R3 → R5 (EIGRP)
 ip policy route-map PBR-EIGRP
```

> The `deny 10.x→10.x` line keeps internal cross-domain traffic out of PBR
> (routed normally). Do NOT leave a plain `set ip next-hop` line — it overrides
> the tracked next-hops and breaks failover.

## 🟤 NAT (Dual-ISP PAT, with internal exclusion)

```cisco
! Inside = internal (R4/R5 side), Outside = ISP side
interface Ethernet0/2
 ip nat inside
interface Ethernet0/3
 ip nat inside
interface Ethernet0/0
 ip nat outside
interface Ethernet0/1
 ip nat outside

! Translate internal 10.x for BOTH ISP exits — exclude internal-to-internal
ip access-list extended NAT-NETS
 deny   ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255   ! 10.x→10.x = NO NAT
 permit ip 10.0.0.0 0.255.255.255 any                       ! 10.x→internet = NAT

route-map NAT-ISP1 permit 10
 match ip address NAT-NETS
 match interface Ethernet0/1
route-map NAT-ISP2 permit 10
 match ip address NAT-NETS
 match interface Ethernet0/0

ip nat inside source route-map NAT-ISP1 interface Ethernet0/1 overload
ip nat inside source route-map NAT-ISP2 interface Ethernet0/0 overload

! After ANY NAT ACL change:
clear ip nat translation *
```

> Private 10.x can't cross the real internet — NAT translates it to R3's
> public ISP-facing IP. One route-map per ISP so both exits translate.
> The `deny 10.x→10.x` line stops internal cross-domain traffic from being
> NATted toward the ISP (the bug that breaks R6↔R8 while routes look fine).


---

## ⚡ IP SLA + Tracking

```cisco
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
```

---

## 🔍 Show Commands

```cisco
! Routing
show ip route
show ip route ospf
show ip route eigrp
show ip route rip
show ip protocols

! Redistribution / loop check
show route-map
show ip ospf database external
show ip eigrp topology

! EIGRP stub transit test
show ip eigrp neighbors detail        ! R8 = "Stub Peer"
show ip eigrp topology all-links      ! shows backup paths (R8 transit)

! PBR
show route-map PBR-OSPF
show route-map PBR-EIGRP
show ip policy

! NAT
show ip nat translations
show ip nat statistics
clear ip nat translation *            ! after any NAT ACL change
debug ip nat                          ! watch what gets translated

! IP SLA / tracking
show ip sla statistics
show ip sla configuration
show track brief
```

---

## 🔬 Failover Test

```cisco
! Confirm normal
R3# show track brief        → both up

! Continuous ping from OSPF client
C> ping 8.8.8.8 -t

! Kill ISP1
R1(config)# interface [internet]
R1(config-if)# shutdown

! Track drops, traffic reroutes to ISP2
R3# show track brief        → Track 1 DOWN, traffic via ISP2 ✅

! Restore
R1(config-if)# no shutdown  → Track 1 up, traffic returns ✅
```

---

## 📝 Redistribution Boundary Summary

| Boundary | Router | Direction | Loop risk? |
|----------|--------|-----------|------------|
| OSPF ↔ EIGRP | R3 | mutual | None (single point) |
| RIP ↔ OSPF | R4 | mutual | None (single point) |
| RIP ↔ EIGRP | R5 | mutual | None (single point) |

> Single router per boundary + separated RIP segments = no loops by
> design. Route tags would only be needed with multiple redistribution
> points between the same two domains.
