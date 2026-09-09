# 6PE Lab with Arista cEOS and Containerlab

A Containerlab topology demonstrating [6PE (RFC 4798)](https://datatracker.ietf.org/doc/html/rfc4798) — IPv6 Provider Edge over MPLS — using Arista cEOS-lab.

IPv6 traffic between customer edge devices is transported across an IPv4-only MPLS core via BGP 6PE, eliminating the need for native IPv6 in the provider network.

## Topology

```
                        ┌────┐
                   ┌────┤ P1 ├────┐
                   │    └────┘    │
                   │              │
 ┌─────┐      ┌────┴─┐        ┌───┴──┐      ┌─────┐
 │ CE1 ├──────┤ PE01 ├─ ─ ─ ─ ┤ PE02 ├──────┤ CE2 │
 └─────┘      └────┬─┘  iBGP  └───┬──┘      └─────┘
   IPv6            │    6PE       │           IPv6
                   │   ┌────┐     │
                   └───┤ P2 ├────-┘
                       └────┘

             ◄── MPLS Core (IS-IS + SR) ──►
```

| Node | Role | Loopback0 | SR Index | Platform |
|------|------|-----------|----------|----------|
| PE01 | Provider Edge | 172.27.0.1/32 | 1 | cEOS-lab |
| PE02 | Provider Edge | 172.27.0.2/32 | 2 | cEOS-lab |
| P1 | Provider | 172.27.0.3/32 | 3 | cEOS-lab |
| P2 | Provider | 172.27.0.4/32 | 4 | cEOS-lab |
| CE1 | Customer Edge | — | — | Linux (Ubuntu) |
| CE2 | Customer Edge | — | — | Linux (Ubuntu) |

## Protocols

| Protocol | Purpose | Details |
|----------|---------|---------|
| IS-IS | IPv4 underlay IGP | Instance `CORE`, area `49.0001`, point-to-point links |
| Segment Routing MPLS | Label distribution | Node-segment indices per router |
| iBGP | IPv6 reachability over MPLS | AS 64496, 6PE address family between PE loopbacks |
| MPLS | Data plane | Label switching across the core |

## IPv6 Addressing (CE-facing)

| Interface | Prefix |
|-----------|--------|
| PE01 Ethernet3 | `fd00:0:0:1::1/64` |
| PE02 Ethernet3 | `fd00:0:0:2::1/64` |
| CE1 eth1 | `fd00:0:0:1::10/64` |
| CE2 eth1 | `fd00:0:0:2::10/64` |

## Prerequisites

- [Containerlab](https://containerlab.dev/) installed
- Arista cEOS-lab image imported as `arista/ceos:latest`

## Quick Start

```bash
# Deploy the lab
sudo containerlab deploy -t topology.clab.yml

# Connect to a node
docker exec -it PE01 Cli

# Destroy the lab
sudo containerlab destroy -t topology.clab.yml
```

## Verification

Once the lab is running, verify 6PE operation from the PE routers:

```
! Check IS-IS adjacencies
show isis neighbors

! Verify MPLS labels
show mpls lfib route

! Check BGP 6PE session
show bgp ipv6 unicast summary

! View IPv6 routes learned via 6PE
show bgp ipv6 unicast

! Test end-to-end IPv6 reachability from CE1
docker exec -it CE1 ping6 fd00:0:0:2::1
docker exec -it CE1 ping6 fd00:0:0:2::10

! Test end-to-end IPv6 reachability from CE2
docker exec -it CE1 ping6 fd00:0:0:1::1
docker exec -it CE1 ping6 fd00:0:0:1::10
```

## How It Works

1. **IS-IS** establishes IPv4 reachability between all routers in the core
2. **Segment Routing MPLS** distributes transport labels — no LDP required
3. **iBGP with 6PE** (RFC 4798) advertises IPv6 prefixes between PE01 and PE02 with an IPv4-mapped IPv6 next-hop, allowing the IPv4 core to forward IPv6 traffic using MPLS labels
4. CE1 and CE2 communicate over IPv6, with traffic MPLS-encapsulated across the provider core

## References

- [RFC 4798 — Connecting IPv6 Islands over IPv4 MPLS Using IPv6 Provider Edge Routers (6PE)](https://datatracker.ietf.org/doc/html/rfc4798)
- [Containerlab Documentation](https://containerlab.dev/)
- [Arista EOS 6PE Configuration Guide](https://www.arista.com/en/um-eos)
