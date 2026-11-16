# DHCP Snooping + Dynamic ARP Inspection — Access-Layer Defense

## Design intent

Two controls deployed as a chain: snooping builds the binding table (trusting only infrastructure uplinks); DAI validates every ARP frame on untrusted ports against that table. Snooping without DAI defends only DHCP; DAI without snooping has nothing to validate against. Evidence from SW2, VLAN 20.

Related screenshots: `12-snooping-binding.png` (refreshed), `13-dai-stats.png` (refreshed), `14-psecure-violation-fa0-6.png`, `15-psecure-fa0-6-status.png`, `16-static-pc-failed-ping.png`, `17-fa0-8-bringup.png`, `18-static-pc-successful-ping.png`, `19-dai-stats-after-static-test.png`

## DHCP Snooping

Enabled on VLANs 10/20/30, uplinks trusted, all access ports untrusted:

```
SW2#show ip dhcp snooping
Switch DHCP snooping is enabled
DHCP snooping is configured on following VLANs:
10,20,30
Insertion of option 82 is disabled
Option 82 on untrusted port is not allowed
Verification of hwaddr field is enabled

Interface                  Trusted    Rate limit (pps)
-----------------------    -------    ----------------
FastEthernet0/5            no         unlimited
FastEthernet0/6            no         unlimited
GigabitEthernet0/1         yes        unlimited
GigabitEthernet0/2         yes        unlimited
```

Populated binding table from legitimate DHCP leases — note there is no entry for the static host (192.168.20.20) used in the controlled test below:

```
SW2#show ip dhcp snooping binding
MacAddress          IpAddress        Lease(sec)  Type           VLAN  Interface
------------------  ---------------  ----------  -------------  ----  -----------------
00:01:C9:78:6C:00   192.168.20.15    0           dhcp-snooping  20    FastEthernet0/6
00:30:A3:54:3C:4E   192.168.20.17    0           dhcp-snooping  20    FastEthernet0/5
Total number of bindings: 2
```

## Dynamic ARP Inspection

Trust model — every host port untrusted, both uplinks trusted:

```
SW2#show ip arp inspection interfaces
Interface        Trust State     Rate(pps)    Burst Interval
---------------  -----------     ---------    --------------
Fa0/1            Untrusted              15                 1
Fa0/2            Untrusted              15                 1
Fa0/3            Untrusted              15                 1
Fa0/4            Untrusted              15                 1
Fa0/5            Untrusted              15                 1
Fa0/6            Untrusted              15                 1
Fa0/7            Untrusted              15                 1
Fa0/8            Untrusted              15                 1
Fa0/9            Untrusted              15                 1
Fa0/10           Untrusted              15                 1
Fa0/11           Untrusted              15                 1
Fa0/12           Untrusted              15                 1
Fa0/13           Untrusted              15                 1
Fa0/14           Untrusted              15                 1
Fa0/15           Untrusted              15                 1
Fa0/16           Untrusted              15                 1
Fa0/17           Untrusted              15                 1
Fa0/18           Untrusted              15                 1
Fa0/19           Untrusted              15                 1
Fa0/20           Untrusted              15                 1
Fa0/21           Untrusted              15                 1
Fa0/22           Untrusted              15                 1
Fa0/23           Untrusted              15                 1
Fa0/24           Untrusted              15                 1
Gig0/1           Trusted                15                 1
Gig0/2           Trusted                15                 1
```

DAI active per-VLAN, with full counters:

```
SW2#show ip arp inspection

Source Mac Validation      : Disabled
Destination Mac Validation : Disabled
IP Address Validation      : Disabled

 Vlan     Configuration    Operation   ACL Match          Static ACL
 ----     -------------    ---------   ---------          ----------
   10     Enabled          Inactive
   20     Enabled          Active
   30     Enabled          Active

 Vlan     ACL Logging      DHCP Logging      Probe Logging
 ----     -----------      ------------      -------------
   10     Deny             Deny              Off
   20     Deny             Deny              Off
   30     Deny             Deny              Off

 Vlan      Forwarded        Dropped     DHCP Drops      ACL Drops
 ----      ---------        -------     ----------      ---------
   10              0              0              0              0
   20              2              0              0              0
   30              0              0              0              0

 Vlan   DHCP Permits    ACL Permits  Probe Permits   Source MAC Failures
 ----   ------------    -----------  -------------   -------------------
   10              0              0              0                     0
   20              2              0              0                     0
   30              0              0              0                     0

 Vlan   Dest MAC Failures   IP Validation Failures   Invalid Protocol Data
 ----   -----------------   ----------------------   ---------------------
   10                   0                        0                       0
   20                   0                        0                       0
   30                   0                        0                       0
```

## Side observation: an unplanned port-security demonstration

While gathering this evidence, a static host (MAC `00E0.F76E.CE28`, IP 192.168.20.18) was connected to **Fa0/6 — a port whose sticky MAC already belonged to another device**. Port Security in `restrict` mode silently dropped all of its traffic while logging every violation:

```
SW2#
%PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred, caused by MAC address 00E0.F76E.CE28 on port FastEthernet0/6.
%PORT_SECURITY-2-PSECURE_VIOLATION: Security violation occurred, caused by MAC address 00E0.F76E.CE28 on port FastEthernet0/6.
```

The host could not ping its gateway (100% loss):

```
C:\>ping 192.168.20.5

Pinging 192.168.20.5 with 32 bytes of data:

Request timed out.
Request timed out.
Request timed out.
Request timed out.

Ping statistics for 192.168.20.5:
    Packets: Sent = 4, Received = 0, Lost = 4 (100% loss),
```

Yet DAI's counters stayed at `Dropped: 0`, confirming the block was Port Security, not ARP inspection. Port state confirms the attribution: the offending MAC is recorded as the last source seen, and the port had accumulated ten violations while remaining administratively up (restrict mode, not shutdown):

```
SW2#show port-security interface fa0/6
Port Security              : Enabled
Port Status                : Secure-down
Violation Mode             : Restrict
Aging Time                 : 0 mins
Aging Type                 : Absolute
SecureStatic Address Aging : Disabled
Maximum MAC Addresses      : 1
Total MAC Addresses        : 1
Configured MAC Addresses   : 0
Sticky MAC Addresses       : 1
Last Source Address:Vlan   : 00E0.F76E.CE28:20
Security Violation Count   : 10
```

(`Secure-down` reflects the host having been unplugged to move it — link state only; the configuration and per-port counters persist.)

An unplanned but clean demonstration that the access-layer controls operate independently: one control (sticky MAC + restrict) silenced the host while logging every event, the other (DAI) kept forwarding valid ARP.

## Controlled test: DAI vs. a static host without a binding

To isolate DAI from Port Security, the static host (IP 192.168.20.20, gateway 192.168.20.5, no DHCP lease — absent from the binding table above) was connected to **Fa0/8**, an unused port with no previously learned sticky MAC, brought into VLAN 20:

```
SW2(config)#int f0/8
SW2(config-if)#no shutdown
SW2(config-if)#switchport mode access
SW2(config-if)#switchport access vlan 20
SW2(config-if)#end

%LINK-5-CHANGED: Interface FastEthernet0/8, changed state to up

%LINEPROTO-5-UPDOWN: Line protocol on Interface FastEthernet0/8, changed state to up
```

The host's ARP for the gateway was **forwarded**, and connectivity succeeded:

```
C:\>ping 192.168.20.5

Pinging 192.168.20.5 with 32 bytes of data:

Reply from 192.168.20.5: bytes=32 time<1ms TTL=255
Reply from 192.168.20.5: bytes=32 time<1ms TTL=255
Reply from 192.168.20.5: bytes=32 time<1ms TTL=255
Reply from 192.168.20.5: bytes=32 time<1ms TTL=255

Ping statistics for 192.168.20.5:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```

Post-test DAI counters — the unbound host's ARP frames were permitted (`DHCP Permits` incremented to 3) despite the binding table containing no entry for it:

```
SW2#show ip arp inspection statistics

 Vlan      Forwarded        Dropped     DHCP Drops     ACL Drops
 ----      ---------        -------     ----------     ---------
   10              0              0              0             0
   20              3              0              0             0
   30              0              0              0             0

 Vlan   DHCP Permits    ACL Permits   Source MAC Failures
 ----   ------------    -----------   -------------------
   10              0              0                     0
   20              3              0                     0
   30              0              0                     0

 Vlan   Dest MAC Failures   IP Validation Failures
 ----   -----------------   ----------------------
   10                   0                        0
   20                   0                        0
   30                   0                        0
```

## Known simulator limitation (confirmed by experiment)

Packet Tracer implements DAI's trust model and counters but does **not** enforce the binding check. The controlled test above is the evidence: a static host with no DHCP binding sent ARP through an untrusted port, the frames were forwarded (`Dropped: 0`, `DHCP Drops: 0`), and the gateway was reachable. On real IOS, GNS3 or EVE-NG, the identical configuration drops those frames (`DHCP Drops >= 1`) and the ping fails at 100%. Readers reproducing this lab on Packet Tracer should treat the trust-model configuration and counters as the evidence of deployment, and validate the actual drop behavior on fuller tooling.
