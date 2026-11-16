# NAT — The Incident: Every Test Passed, Half the Flows Were Broken

Related screenshots: `08-nat-translations-postfix.png`, `09-nat-statistics.png`, `10-isp-route.png`, `11-isp-ping-fail.png`

## Symptom

All VLANs reached the internet; every ping passed. Nothing looked wrong.

## Discovery

While verifying translations (not connectivity), `show ip nat translations` on R1 showed flows from only *some* VLANs.

Root cause chain:

1. MLSW2 load-balances edge-bound traffic across two equal-cost OSPF paths (evidence: `ospf.md`) — one exits via MLSW1's direct link to R1, the other via R2 into R1's Gi0/0/2.
2. Flows that selected the R2 path arrived at R1 on **Gi0/0/2 — which was missing `ip nat inside`**.
3. Those flows left R1 **untranslated**, private source addresses intact.

A client-side traceroute (in `connectivity-tests.md`) shows a VLAN 20 flow taking exactly this second branch: hop 2 is `192.168.1.9` (R2), entering R1 on Gi0/0/2.

## Why it stayed hidden

The simulated ISP had a static route for `192.168.0.0/16` pointing back toward R1, so untranslated packets still got replies. In production, no ISP routes RFC 1918 space — those packets die at the edge. The static route was an artificial kindness masking broken NAT.

> **Evidence note:** the pre-fix state (missing `ip nat inside` on Gi0/0/2, and the partial translation table that led to the discovery) was identified and remediated during the same session, before this repository's evidence discipline was established, and therefore was not captured. The pre-fix configuration described here is reconstructed from the discovery session. All post-fix evidence below was captured from the live, corrected topology.

## Fix, part 1 — tag every inside-facing interface

`ip nat inside` applied to **all** inside interfaces of R1. With ECMP, one missing tag silently untranslates half the flows.

Statistics confirming both inside interfaces are registered with NAT (current, fixed state):

```
R1#show ip nat statistics
Total translations: 1 (0 static, 1 dynamic, 1 extended)
Outside Interfaces: GigabitEthernet0/0/1
Inside Interfaces: GigabitEthernet0/0/0 , GigabitEthernet0/0/2
Hits: 598  Misses: 38
Expired translations: 1
Dynamic mappings:
```

Both inside interfaces are listed: Gi0/0/0 **and** Gi0/0/2.

Current interface state on the two inside-facing links (addresses confirm the ECMP path arithmetic in `ospf.md`):

```
R1#show interface gigabitEthernet 0/0/0
GigabitEthernet0/0/0 is up, line protocol is up (connected)
  Hardware is ISR4331-3x1GE, address is 0001.428a.cb01 (bia 0001.428a.cb01)
  Internet address is 192.168.1.5/30
  MTU 1500 bytes, BW 1000000 Kbit, DLY 10 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation ARPA, loopback not set
  Keepalive not supported
  Full Duplex, 100Mbps, link type is auto, media type is Auto Select
  output flow-control is on, input flow-control is on
  ARP type: ARPA, ARP Timeout 04:00:00,
  Last input 00:00:08, output 00:00:05, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/375/0 (size/max/drops); Total output drops: 0
  Queueing strategy: fifo
  Output queue :0/40 (size/max)
  5 minute input rate 80 bits/sec, 0 packets/sec
  5 minute output rate 71 bits/sec, 0 packets/sec
     137 packets input, 14472 bytes, 0 no buffer
     Received 0 broadcasts (0 IP multicasts)
     0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
     0 watchdog, 1017 multicast, 0 pause input
     0 input packets with dribble condition detected
     86 packets output, 11190 bytes, 0 underruns
     0 output errors, 0 collisions, 1 interface resets
     0 unknown protocol drops
     0 babbles, 0 late collision, 0 deferred
     0 lost carrier, 0 no carrier
     0 output buffer failures, 0 output buffers swapped out
```

```
R1#show interface gigabitEthernet 0/0/2
GigabitEthernet0/0/2 is up, line protocol is up (connected)
  Hardware is ISR4331-3x1GE, address is 0001.428a.cb03 (bia 0001.428a.cb03)
  Internet address is 192.168.1.17/30
  MTU 1500 bytes, BW 1000000 Kbit, DLY 10 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation ARPA, loopback not set
  Keepalive not supported
  output flow-control is on, input flow-control is on
  ARP type: ARPA, ARP Timeout 04:00:00,
  Last input 00:00:08, output 00:00:05, output hang never
  Last clearing of "show interface" counters never
  Input queue: 0/375/0 (size/max/drops); Total output drops: 0
  Queueing strategy: fifo
  Output queue :0/40 (size/max)
  5 minute input rate 43 bits/sec, 0 packets/sec
  5 minute output rate 44 bits/sec, 0 packets/sec
     46 packets input, 5500 bytes, 0 no buffer
     Received 0 broadcasts (0 IP multicasts)
     0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
     0 watchdog, 1017 multicast, 0 pause input
     0 input packets with dribble condition detected
     34 packets output, 4132 bytes, 0 underruns
     0 output errors, 0 collisions, 1 interface resets
     0 unknown protocol drops
     0 babbles, 0 late collision, 0 deferred
     0 lost carrier, 0 no carrier
     0 output buffer failures, 0 output buffers swapped out
```

Post-fix translation table after actively pinging the ISP from **PC1 (VLAN 10)**, **PC3 (VLAN 20)** and **PC5 (VLAN 30)**:

```
R1#show ip nat translations
Pro  Inside global     Inside local       Outside local      Outside global
icmp 192.168.1.14:1024 192.168.20.17:9    209.165.201.1:9    209.165.201.1:1024
icmp 192.168.1.14:1025 192.168.30.16:9    209.165.201.1:9    209.165.201.1:1025
icmp 192.168.1.14:1026 192.168.30.16:10   209.165.201.1:10   209.165.201.1:1026
icmp 192.168.1.14:1027 192.168.30.16:11   209.165.201.1:11   209.165.201.1:1027
icmp 192.168.1.14:1028 192.168.30.16:12   209.165.201.1:12   209.165.201.1:1028
icmp 192.168.1.14:1029 192.168.20.17:10   209.165.201.1:10   209.165.201.1:1029
icmp 192.168.1.14:1030 192.168.20.17:11   209.165.201.1:11   209.165.201.1:1030
icmp 192.168.1.14:1031 192.168.20.17:12   209.165.201.1:12   209.165.201.1:1031
icmp 192.168.1.14:10   192.168.10.17:10   209.165.201.1:10   209.165.201.1:10
icmp 192.168.1.14:11   192.168.10.17:11   209.165.201.1:11   209.165.201.1:11
icmp 192.168.1.14:12   192.168.10.17:12   209.165.201.1:12   209.165.201.1:12
icmp 192.168.1.14:9    192.168.10.17:9    209.165.201.1:9    209.165.201.1:9
udp  192.168.1.14:123  192.168.1.14:123   209.165.201.1:123  209.165.201.1:123
```

This is the decisive evidence for the fix: flows from **VLAN 10 (PC1, 192.168.10.17), VLAN 20 (PC3, 192.168.20.17) and VLAN 30 (PC5, 192.168.30.16)** all appear in the table, each translated to the same inside-global address **192.168.1.14** (PAT distinguishing flows by port), alongside the standing NTP flow from R1's loopback. Before the fix, flows arriving on the untagged Gi0/0/2 never appeared here at all — which is precisely why inspecting *translations*, not connectivity, was the discovery step.

Supporting connectivity at capture time — **PC1, VLAN 10** (192.168.10.17):

```
C:\>ping 209.165.201.1

Pinging 209.165.201.1 with 32 bytes of data:

Reply from 209.165.201.1: bytes=32 time<1ms TTL=125
Reply from 209.165.201.1: bytes=32 time<1ms TTL=125
Reply from 209.165.201.1: bytes=32 time<1ms TTL=125
Reply from 209.165.201.1: bytes=32 time<1ms TTL=125

Ping statistics for 209.165.201.1:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```

The reply **TTL of 125** from a client starting at TTL 128 means three router decrements (MLS switch → R1 → ISP), consistent with the exit via MLSW1's direct link to R1. Flows taking the other ECMP branch (via R2) would arrive with TTL 124. Whether a pre-fix flow was broken depended on which exit it selected — which is why the symptoms were partial rather than absolute. *(TTL interpretation is inferred from the topology, not stated by the device.)*

## Fix, part 2 — remove the crutch

The ISP's static route to `192.168.0.0/16` was removed entirely. A correctly operating NAT requires zero reachability into private space:

```
ISP#show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default, U - per-user static route, o - ODR
       P - periodic downloaded static route

Gateway of last resort is not set

     192.168.1.0/24 is variably subnetted, 2 subnets, 2 masks
C       192.168.1.12/30 is directly connected, GigabitEthernet0/1
L       192.168.1.13/32 is directly connected, GigabitEthernet0/1
     209.165.201.0/24 is variably subnetted, 2 subnets, 2 masks
C       209.165.201.0/30 is directly connected, GigabitEthernet0/0
L       209.165.201.2/32 is directly connected, GigabitEthernet0/0
```

Only the two connected networks remain — no route to RFC 1918 space, no gateway of last resort — and connectivity continues to work.

## Unsolicited inbound fails — by design

```
ISP#ping 192.168.10.15

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.10.15, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)
```

An unsolicited inbound failure is not a bug here; it is the feature working. Private hosts are unreachable from outside without a translation, exactly as intended.

## Lesson

Connectivity tests prove packets flow. Only translations prove NAT works. A test environment can hide a real configuration fault if it provides routing behavior that would not exist in production — and the fix isn't more pinging; it's removing the crutch and retesting (retest evidence: `connectivity-tests.md`).
