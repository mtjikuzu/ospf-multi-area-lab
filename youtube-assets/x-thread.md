# X/Twitter Thread — Multi-Area OSPF Lab

Copy each numbered section as a separate tweet. Post as a thread (reply to previous).

---

## 1/15 — Hook

I configured multi-area OSPF across 5 Cisco routers from scratch.

5 routers. 3 areas. 14 adjacencies. All live on real hardware.

Here's everything I learned, in one thread.

---

## 2/15 — What is OSPF

OSPF is a link-state routing protocol.

Unlike distance-vector protocols, every router builds a complete map of the network and computes the shortest path.

For large networks, OSPF divides the topology into areas to keep things manageable.

---

## 3/15 — Why Multi-Area

Single-area OSPF works fine for small networks.

But as you grow:
- Link-state database gets huge
- SPF recalculations slow down
- Every topology change floods everywhere

Multi-area fixes this. Each area maintains its own database. Area Border Routers summarize between them.

---

## 4/15 — The Topology

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

R1 = backbone core
R2 = ABR (Area 0 ↔ Area 1)
R3 = ABR (Area 0 ↔ Area 2)
R4, R5 = distribution routers

---

## 5/15 — The Design Rules

Three rules I followed:

1. Every area connects to Area 0 (backbone)
2. ABRs have interfaces in 2+ areas
3. Every area has a backup path to the backbone

That cross-link between R4 and R5? It's there so if R2 fails, Area 1 can still reach the backbone through R5 → R3.

---

## 6/15 — R1 Config (Backbone Core)

```
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 ip ospf 1 area 0

interface Ethernet0/1
 ip address 10.0.12.1 255.255.255.252
 ip ospf network point-to-point
 ip ospf 1 area 0

router ospf 1
 router-id 1.1.1.1
```

Always set router-id explicitly. If you rely on auto-selection, it changes when interfaces flap.

---

## 7/15 — Point-to-Point Network Type

Every inter-router link uses `ip ospf network point-to-point`.

Why? It skips DR/BDR election.

On a point-to-point link there are only 2 routers. Electing a Designated Router is pointless overhead. Adjacency forms faster.

---

## 8/15 — R2 Config (The ABR)

R2 has interfaces in Area 0 AND Area 1:

```
interface Ethernet0/1
 ip ospf 1 area 0    ← to R1

interface Ethernet0/3
 ip ospf 1 area 1    ← to R4
```

That single area assignment on Eth0/3 is what makes R2 an ABR.

It generates Type 3 Summary LSAs to advertise routes between areas.

---

## 9/15 — R5 Config (The Clever One)

R5 is mostly Area 2, but the cross-link to R4 is in Area 1:

```
interface Ethernet0/1
 ip ospf 1 area 2    ← to R3

interface Ethernet0/2
 ip ospf 1 area 1    ← to R4 (cross-link)

interface Ethernet0/3
 ip ospf 1 area 2    ← to SW2
```

This makes R5 an ABR too. If R3 goes down, traffic flows R5 → R4 → R2 → backbone.

---

## 10/15 — Watching Convergence

After configuring all 5 routers, I enabled OSPF adjacency debug on R1:

```
debug ip ospf adj
```

Watched the state machine live:

Init → 2-Way → ExStart → Exchange → Loading → FULL

Once all neighbors hit FULL, convergence is complete.

---

## 11/15 — Verifying Neighbors

```
R1# show ip ospf neighbor

Neighbor ID  State      Interface
2.2.2.2      FULL/ -    Ethernet0/1
3.3.3.3      FULL/ -    Ethernet0/2
```

FULL = databases synchronized.
The dash after FULL = no DR election (point-to-point).

Every adjacency across all 5 routers: FULL.

---

## 12/15 — Reading the Routing Table

```
R1# show ip route ospf

O     2.2.2.2 [110/11] via 10.0.12.2
O     3.3.3.3 [110/11] via 10.0.13.2
O IA  4.4.4.4 [110/21] via 10.0.12.2
O IA  5.5.5.5 [110/21] via 10.0.13.2
```

O = intra-area (same area)
O IA = inter-area (learned from ABR)

R1 sees R4 and R5 as inter-area routes. Multi-area OSPF is working.

---

## 13/15 — Key Takeaways

What I'd tell myself before building this:

→ Set router-id explicitly (don't rely on auto)
→ Use point-to-point on all router-to-router links
→ Design for redundancy (every area needs a backup path to Area 0)
→ ABR = interfaces in 2+ areas (it's that simple)
→ O IA routes prove your multi-area design works

---

## 14/15 — The Lab Stack

Everything runs on one Linux server:

- Cisco IOL routers via ContainerLab
- Remotion for animated intro/outro
- Python for automated recording

Total memory: ~4 GB for all 5 routers.

No cloud, no GNS3, no Packet Tracer. Just containers.

---

## 15/15 — Resources

Full video walkthrough (16 min, every command typed live):
[YouTube link]

All router configs + ContainerLab topology:
github.com/mtjikuzu/ospf-multi-area-lab

Like this thread? Follow @TheTechHuddle for more networking labs.

#OSPF #Cisco #CCNA #Networking #ContainerLab

---

# Posting Notes

- Thread length: 15 tweets
- Post tweet 1 first, then reply to it with tweet 2, reply to tweet 2 with tweet 3, etc.
- Add the YouTube link in tweet 15 once the video is uploaded
- Optional: attach a screenshot of the topology or terminal output to tweets 4, 6, 11, or 12 for visual engagement
- Best posting times for tech content: Tuesday-Thursday, 8-10am or 12-1pm ET
- After posting, quote-tweet your own tweet 1 with a one-liner hook to boost visibility
