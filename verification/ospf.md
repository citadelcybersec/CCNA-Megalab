# OSPF — Adjacencies and ECMP Verification

## Design intent

Single Area 0. Loopbacks (1.1.1.1, 2.2.2.2, 3.3.3.3) anchor the OSPF router IDs. User-facing SVIs and loopbacks are passive; only transit links form adjacencies. By design, MLSW2 has two equal-cost paths toward the edge: via the EtherChannel to MLSW1, and via R2 — genuine per-flow load balancing, which is central to the NAT incident (see `nat.md`).

Related screenshots: `02-etherchannel.png`, `05-ospf-neighbor.png`, `06-ospf-route-ecmp.png`

## EtherChannel — the transport path

```
MLSW1#show etherchannel summary
Flags:  D - down        P - bundled in port-channel
        I - stand-alone s - suspended
        H - Hot-standby (LACP only)
        R - Layer3      S - Layer2
        U - in use      f - failed to allocate aggregator
        u - unsuitable for bundling
        w - waiting to be aggregated
        d - default port

Number of channel-groups in use: 1
Number of aggregators: 1

Group  Port-channel  Protocol    Ports
------+-------------+-----------+----------------------------------------
1      Po1(RU)       LACP        Gig1/3(P)  Gig1/4(P)
```

`Po1(RU)` with both members `(P)`: the LACP bundle is Layer 2, in use, with both physical links bundled.

## Adjacencies

MLSW2 has FULL adjacencies over the EtherChannel (toward MLSW1, router ID 3.3.3.3) and via its direct link to R2 (router ID 2.2.2.2):

```
MLSW2#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
2.2.2.2          1    FULL/BDR        00:00:39    192.168.1.9     GigabitEthernet1/8
3.3.3.3          1    FULL/BDR        00:00:39    192.168.1.1     Port-channel1
```

## ECMP evidence

MLSW2's OSPF routing table — the default route (`O*E2 0.0.0.0/0`) and R1's loopback (1.1.1.1) each install **two** equal-cost next-hops with identical `[110/...]` metrics:

```
MLSW2#show ip route ospf
     1.0.0.0/32 is subnetted, 1 subnets
O       1.1.1.1 [110/3] via 192.168.1.9, 00:18:01, GigabitEthernet1/8
                [110/3] via 192.168.1.1, 00:18:01, Port-channel1
     2.0.0.0/32 is subnetted, 1 subnets
O       2.2.2.2 [110/2] via 192.168.1.9, 00:18:01, GigabitEthernet1/8
     3.0.0.0/32 is subnetted, 1 subnets
O       3.3.3.3 [110/2] via 192.168.1.1, 00:18:01, Port-channel1
     192.168.1.0/24 is variably subnetted, 6 subnets, 2 masks
O       192.168.1.4  [110/2] via 192.168.1.1, 00:18:01, Port-channel1
O       192.168.1.16 [110/2] via 192.168.1.9, 00:18:01, GigabitEthernet1/8
O*E2 0.0.0.0/0 [110/1] via 192.168.1.9, 00:18:01, GigabitEthernet1/8
O*E2 0.0.0.0/0 [110/1] via 192.168.1.1, 00:18:01, Port-channel1
```

Reading it:

- `0.0.0.0/0` and `1.1.1.1`: two next-hops each — **via 192.168.1.1 (Port-channel1, toward MLSW1)** and **via 192.168.1.9 (Gi1/8, toward R2)** — same metric, different egress. Per-flow ECMP.
- `2.2.2.2` (R2's loopback): only via R2 — directly reachable through that path, so no equal-cost alternative exists.
- `3.3.3.3` and `192.168.1.4` (MLSW1 side): only via the EtherChannel — the mirror-image reason.

R1's interface addresses corroborate the path arithmetic: `show interface gigabitEthernet 0/0/0` on R1 shows `Internet address is 192.168.1.5/30` (the `192.168.1.4/30` subnet MLSW2 reaches via MLSW1), and Gi0/0/2 shows `192.168.1.17/30` — the `192.168.1.16/30` subnet MLSW2 reaches via R2. Both outputs are reproduced in `nat.md`.

Consequence: roughly half of MLSW2's edge-bound flows traverse the path through R2. That is why a missing `ip nat inside` on R1's second inside-facing interface could silently untranslate those flows while every connectivity test passed — see `nat.md`. A client-side confirmation of the R2 branch appears in `connectivity-tests.md`: a traceroute from a VLAN 20 PC shows hop 2 as `192.168.1.9` (R2).
