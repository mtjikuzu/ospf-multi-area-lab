# Multi-Area OSPF from Scratch: 5 Routers, 3 Areas, One Lab

I built a multi-area OSPF network with 5 Cisco routers, 3 OSPF areas, and 14 adjacencies — all running on a single Linux server. Every command typed live, nothing simulated.

This is the full walkthrough. By the end, you'll understand how multi-area OSPF works, why it matters, and how to build one yourself.

---

## What is OSPF?

OSPF (Open Shortest Path First) is a link-state routing protocol. Every router builds a complete map of the network topology, then computes the shortest path to every destination using Dijkstra's algorithm.

This is fundamentally different from distance-vector protocols like RIP, where routers only know what their neighbors tell them. In OSPF, every router has the full picture.

But that full picture comes at a cost.

## Why Multi-Area?

In a single-area OSPF deployment, every router maintains an identical Link-State Database (LSDB). Every topology change — a link going down, a new route appearing — triggers a Shortest Path First recalculation on every router in the area.

For small networks, this is fine. For large ones, it becomes a problem:

- The LSDB grows with every router and link added
- SPF recalculations consume CPU time
- Flooding LSAs across hundreds of routers wastes bandwidth

Multi-area OSPF solves this by dividing the network into areas. Each area maintains its own LSDB. Routers only need the full topology of their own area. Routes from other areas arrive as summarized entries through Area Border Routers (ABRs).

Result: smaller databases, faster convergence, less CPU overhead.

---

## The Topology

Here's what we're building:

```
            Area 0 (Backbone)
        R1 ──── R2 ──── R3
        │       │       │
        └───────┼───────┘
                │
           ┌────┴────┐
           R4        R5
         Area 1    Area 2
           │         │
          SW1       SW2
```

Five routers, two switches, three OSPF areas:

**R1** — Backbone core router. All interfaces in Area 0. Router ID: 1.1.1.1

**R2** — Area Border Router. Bridges Area 0 and Area 1. Router ID: 2.2.2.2

**R3** — Area Border Router. Bridges Area 0 and Area 2. Router ID: 3.3.3.3

**R4** — Distribution router in Area 1. Router ID: 4.4.4.4

**R5** — Distribution router in Area 2, with a strategic cross-link to R4. Router ID: 5.5.5.5

### Subnet Addressing

| Link | Subnet | Area |
|------|--------|------|
| R1 ↔ R2 | 10.0.12.0/30 | 0 |
| R1 ↔ R3 | 10.0.13.0/30 | 0 |
| R2 ↔ R3 | 10.0.23.0/30 | 0 |
| R2 ↔ R4 | 10.0.24.0/30 | 1 |
| R3 ↔ R5 | 10.0.35.0/30 | 2 |
| R4 ↔ R5 | 10.0.45.0/30 | 1 |
| R4 ↔ SW1 | 10.0.41.0/30 | 1 |
| R5 ↔ SW2 | 10.0.52.0/30 | 2 |

---

## Three Design Rules

Before touching a router, I committed to three rules:

**1. Every area connects to Area 0.**
Area 0 is the backbone. All inter-area traffic flows through it. This is a hard requirement in OSPF — non-backbone areas cannot talk to each other directly.

**2. ABRs have interfaces in two or more areas.**
An Area Border Router is simply a router with interfaces in multiple areas. It maintains separate LSDBs for each area and generates Type 3 Summary LSAs to advertise routes between them.

**3. Every area has a backup path to the backbone.**
The R4 ↔ R5 cross-link exists for redundancy. If R2 fails, Area 1 traffic can still reach the backbone through R5 → R3.

---

## Configuration

### R1 — Backbone Core

R1 is straightforward. Every interface lives in Area 0.

```
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 ip ospf 1 area 0

interface Ethernet0/1
 ip address 10.0.12.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0

interface Ethernet0/2
 ip address 10.0.13.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0

router ospf 1
 router-id 1.1.1.1
```

Two things to note:

**Set router-id explicitly.** If you rely on the automatic selection (highest loopback IP, then highest physical IP), the router ID can change when interfaces go up and down. That causes adjacency flaps. Always pin it.

**Use point-to-point network type.** Every inter-router link in this lab uses `ip ospf network point-to-point`. On a link with only two routers, there's no reason to elect a Designated Router (DR) and Backup DR (BDR). Point-to-point skips that election entirely, forming adjacencies faster.

### R2 — The ABR (Area 0 ↔ Area 1)

R2 is where multi-area OSPF comes alive.

```
interface Ethernet0/1
 ip address 10.0.12.2 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0

interface Ethernet0/2
 ip address 10.0.23.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0

interface Ethernet0/3
 ip address 10.0.24.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 1
```

Look at Ethernet0/3: `ip ospf 1 area 1`. That one line is the area boundary. Everything else on R2 is Area 0. Ethernet0/3 is Area 1.

R2 now maintains two separate Link-State Databases — one for Area 0, one for Area 1. It generates Type 3 Summary LSAs to advertise Area 0 routes into Area 1 and vice versa.

Routers in Area 1 don't need to know the full Area 0 topology. They just receive summarized routes through R2. That's the scalability benefit of multi-area OSPF.

### R3 — The ABR (Area 0 ↔ Area 2)

R3 mirrors R2, bridging Area 0 to Area 2 instead of Area 1. Same pattern — backbone interfaces in Area 0, one interface to R5 in Area 2.

Notice the symmetry. Both ABRs maintain full backbone connectivity. If one ABR fails, the other still bridges its area to the backbone.

### R5 — The Clever One

R5 is the most interesting router in this design.

```
interface Ethernet0/1
 ip ospf 1 area 2    ← uplink to R3

interface Ethernet0/2
 ip ospf 1 area 1    ← cross-link to R4

interface Ethernet0/3
 ip ospf 1 area 2    ← downlink to SW2
```

R5 is mostly Area 2, but the cross-link to R4 is placed in **Area 1**. This makes R5 an ABR as well, sitting between Area 1 and Area 2.

Why? Redundancy. If R3 goes down, R5 can still reach the backbone:

R5 → R4 (Area 1) → R2 (ABR) → Area 0

Every area maintains a backup path to the backbone. That's resilient design.

---

## Convergence

After configuring all five routers, I enabled OSPF adjacency debugging on R1:

```
R1# debug ip ospf adj
```

Then watched the state machine progress in real time:

**Init** → Received a Hello from a neighbor
**2-Way** → Bidirectional communication established
**ExStart** → Negotiating master/slave for database exchange
**Exchange** → Trading Database Description packets
**Loading** → Requesting missing LSAs
**Full** → Databases synchronized. Adjacency complete.

Once all neighbors reach Full state, the OSPF mesh is converged. Every router has the complete topology of its area.

---

## Verification

### Checking Neighbors

```
R1# show ip ospf neighbor

Neighbor ID  Pri  State       Dead Time  Address       Interface
2.2.2.2        0  FULL/ -     00:00:32   10.0.12.2     Ethernet0/1
3.3.3.3        0  FULL/ -     00:00:39   10.0.13.2     Ethernet0/2
```

Both neighbors in FULL state. The `FULL/ -` (dash) means no DR election happened — exactly what we expect on point-to-point links.

R2 (the ABR) shows three FULL neighbors: R1, R3 in Area 0, and R4 in Area 1. It sits at the boundary, maintaining separate databases for each area.

Every adjacency across all five routers: FULL.

### Reading the Routing Table

This is where you see multi-area OSPF in action:

```
R1# show ip route ospf

O     2.2.2.2 [110/11] via 10.0.12.2, Ethernet0/1
O     3.3.3.3 [110/11] via 10.0.13.2, Ethernet0/2
O IA  4.4.4.4 [110/21] via 10.0.12.2, Ethernet0/1
O IA  5.5.5.5 [110/21] via 10.0.13.2, Ethernet0/2
O     10.0.23.0/30 [110/20] via 10.0.13.2, Ethernet0/2
O IA  10.0.24.0/30 [110/20] via 10.0.12.2, Ethernet0/1
O IA  10.0.35.0/30 [110/20] via 10.0.13.2, Ethernet0/2
```

**O** = Intra-area route (learned within Area 0)
**O IA** = Inter-area route (learned from another area via an ABR)

R1 sees R2 and R3 as intra-area (same area). R4 and R5 show up as inter-area routes — they're in Area 1 and Area 2 respectively, reachable through the ABRs.

All subnets reachable. The multi-area design is working exactly as intended.

---

## The Lab Stack

Everything runs on a single Linux server. No cloud, no GNS3, no Packet Tracer.

**Cisco IOL** (IOS on Linux) — Lightweight Cisco router images running as native Linux binaries inside containers. Each router uses about 200-400 MB of RAM.

**ContainerLab** — Defines the topology in YAML, spins up all routers and wires them together with a single command: `sudo containerlab deploy -t topology.yml`

**Remotion** — React-based video framework for the animated intro and outro scenes.

Total memory for all 5 routers: ~2 GB. With KSM (Kernel Same-page Merging), identical memory pages across IOL instances get deduplicated, dropping actual usage even further.

---

## Key Takeaways

If you're building multi-area OSPF for the first time:

→ **Set router-id explicitly.** Don't rely on auto-selection. Pin it to a loopback address.

→ **Use point-to-point on router-to-router links.** Skip the unnecessary DR/BDR election.

→ **Every non-backbone area must connect to Area 0.** This is an OSPF requirement, not a suggestion.

→ **An ABR is just a router with interfaces in 2+ areas.** One interface assignment is all it takes.

→ **Design for redundancy.** Every area should have a backup path to the backbone. One ABR failure shouldn't isolate an entire area.

→ **O IA routes in your routing table prove it works.** If you see inter-area routes, your multi-area design is operational.

---

## Resources

Full video walkthrough — 16 minutes, every command typed live on real routers:
[YouTube link]

All router configs + ContainerLab topology on GitHub:
github.com/mtjikuzu/ospf-multi-area-lab

---

*Built by The Tech Huddle. Follow for more hands-on networking labs.*
