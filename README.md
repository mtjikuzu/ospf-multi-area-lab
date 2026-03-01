# Multi-Area OSPF Configuration Lab

Router configurations and ContainerLab topology for a multi-area OSPF lab with 5 Cisco IOL routers across 3 OSPF areas.

Companion repo for the [video walkthrough on YouTube](https://youtube.com/@TheTechHuddle).

## Topology

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

## Routers

| Router | Role | Router ID | OSPF Areas | Description |
|--------|------|-----------|------------|-------------|
| **R1** | Core | 1.1.1.1 | Area 0 | Backbone internal router |
| **R2** | ABR | 2.2.2.2 | Area 0, Area 1 | Area Border Router — backbone to Area 1 |
| **R3** | ABR | 3.3.3.3 | Area 0, Area 2 | Area Border Router — backbone to Area 2 |
| **R4** | Distribution | 4.4.4.4 | Area 1 | Distribution router, cross-linked to R5 |
| **R5** | Distribution | 5.5.5.5 | Area 2 | Distribution router, cross-linked to R4 |

## Subnets

| Link | Subnet | OSPF Area |
|------|--------|-----------|
| R1 ↔ R2 | 10.0.12.0/30 | Area 0 |
| R1 ↔ R3 | 10.0.13.0/30 | Area 0 |
| R2 ↔ R3 | 10.0.23.0/30 | Area 0 |
| R2 ↔ R4 | 10.0.24.0/30 | Area 1 |
| R3 ↔ R5 | 10.0.35.0/30 | Area 2 |
| R4 ↔ R5 | 10.0.45.0/30 | Area 1 |
| R4 ↔ SW1 | 10.0.41.0/30 | Area 1 |
| R5 ↔ SW2 | 10.0.52.0/30 | Area 2 |

## Files

```
configs/
  R1.cfg          # Backbone core router
  R2.cfg          # ABR — Area 0 ↔ Area 1
  R3.cfg          # ABR — Area 0 ↔ Area 2
  R4.cfg          # Distribution — Area 1
  R5.cfg          # Distribution — Area 2

topology/
  5-node-ospf.clab.yml   # ContainerLab topology definition
```

## Running the Lab

### Prerequisites

- [ContainerLab](https://containerlab.dev/) installed
- Cisco IOL image: `vrnetlab/cisco_iol:17.12.01` (requires [vrnetlab](https://github.com/srl-labs/vrnetlab) build)

### Deploy

```bash
sudo containerlab deploy -t topology/5-node-ospf.clab.yml
```

### Access Routers

```bash
ssh admin@localhost -p 22401   # R1
ssh admin@localhost -p 22402   # R2
ssh admin@localhost -p 22403   # R3
ssh admin@localhost -p 22404   # R4
ssh admin@localhost -p 22405   # R5
```

Default credentials: `admin` / `admin`

### Apply Configs

You can paste configurations manually or use a script:

```bash
for i in 1 2 3 4 5; do
  sshpass -p admin ssh -o StrictHostKeyChecking=no -p 2240${i} admin@localhost \
    "configure terminal" < configs/R${i}.cfg
done
```

### Verify

```bash
# Check OSPF neighbors
ssh admin@localhost -p 22401 "show ip ospf neighbor"

# Check routing table
ssh admin@localhost -p 22401 "show ip route ospf"
```

### Teardown

```bash
sudo containerlab destroy -t topology/5-node-ospf.clab.yml
```

## Key Concepts

- **Multi-area OSPF** — Backbone (Area 0) with two non-backbone areas for scalability
- **Area Border Routers (ABR)** — R2 and R3 have interfaces in multiple areas
- **Point-to-point network type** — Used on all inter-router links (no DR/BDR election)
- **Router ID via Loopback0** — Deterministic RID assignment for stable OSPF operation
- **Inter-area routes (O IA)** — Routes learned from other areas via ABR summarization
- **Cross-link redundancy** — R4 ↔ R5 link provides path diversity between areas

## License

MIT
