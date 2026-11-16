# IP Addressing Table

## Point-to-point infrastructure links (/30)

| Network | Link | Endpoint A | Endpoint B |
|---|---|---|---|
| 192.168.1.0/30 | Inter-switch routed link | MLSW1 Po1 — 192.168.1.1 | MLSW2 Po1 — 192.168.1.2 |
| 192.168.1.4/30 | MLSW1 ↔ R1 | MLSW1 Gi1/8 — 192.168.1.6 | R1 Gi0/0/0 — 192.168.1.5 |
| 192.168.1.8/30 | MLSW2 ↔ R2 | MLSW2 Gi1/8 — 192.168.1.10 | R2 Gi0/0/0 — 192.168.1.9 |
| 192.168.1.12/30 | R1 ↔ ISP | R1 Gi0/0/1 — 192.168.1.14 (NAT outside) | ISP Gi0/1 — 192.168.1.13 |
| 192.168.1.16/30 | R1 ↔ R2 | R1 Gi0/0/2 — 192.168.1.17 (NAT inside) | R2 Gi0/0/2 — 192.168.1.18 |
| 209.165.201.0/30 | ISP ↔ external | ISP Gi0/0 — 209.165.201.2 | Server0 — 209.165.201.1 |

## User VLANs and HSRP gateways

| VLAN | Subnet | MLSW1 SVI | MLSW2 SVI | HSRP VIP (gateway) | Active | DHCP pool on R1 |
|---|---|---|---|---|---|---|
| 10 | 192.168.10.0/24 | 192.168.10.3 | 192.168.10.4 | 192.168.10.5 | MLSW1 | Excluded .1–.9 |
| 20 | 192.168.20.0/24 | 192.168.20.3 | 192.168.20.4 | 192.168.20.5 | MLSW2 | Excluded .1–.9 |
| 30 | 192.168.30.0/24 | 192.168.30.3 | 192.168.30.4 | 192.168.30.5 | MLSW1 | Excluded .1–.9 |
| 99 | — (native, no hosts) | — | — | — | — | — |

## Access switch management SVIs

| Switch | SVI | Address | Notes |
|---|---|---|---|
| SW1 | Vlan10 | 192.168.10.6 | Default gateway 192.168.10.5 (HSRP VIP) |
| SW2 | Vlan20 | 192.168.20.6 | Default gateway 192.168.20.5 (HSRP VIP) |
| SW3 | Vlan30 | 192.168.30.6 | Default gateway 192.168.30.5 (HSRP VIP) |

## Loopbacks / OSPF router IDs

| Device | Loopback0 | OSPF Router-ID |
|---|---|---|
| R1 | 1.1.1.1/32 | 1.1.1.1 (also internal NTP source) |
| R2 | 2.2.2.2/32 | 2.2.2.2 |
| MLSW1 | 3.3.3.3/32 | 3.3.3.3 |
| MLSW2 | 4.4.4.4/32 | 4.4.4.4 |

## Services

| Service | Provider | Address | Notes |
|---|---|---|---|
| DHCP | R1 (centralized) | 192.168.1.5 (Gi0/0/0) | Relay via `ip helper-address 192.168.1.5` on all SVIs |
| DNS | Server0 | 209.165.201.1 | Handed out in all DHCP pools |
| NTP (internal) | R1 loopback | 1.1.1.1 | All infrastructure devices sync here |
| NTP (upstream) | Server0 | 209.165.201.1 | R1 syncs here; `ntp master 3` fallback |
| NAT/PAT | R1 Gi0/0/1 | Outside address 192.168.1.14 | `NAT-ALLOWED` eligibility ACL |
