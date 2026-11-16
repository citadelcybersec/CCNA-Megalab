# DHCP Snooping + Dynamic ARP Inspection — Access-Layer Defense

## Design intent

Two controls deployed as a chain: snooping builds the binding table (trusting only infrastructure uplinks); DAI validates every ARP frame on untrusted ports against that table. Snooping without DAI defends only DHCP; DAI without snooping has nothing to validate against. Evidence from SW2, VLAN 20.

Related screenshots: `12-snooping-binding.png`, `13-dai-stats.png`

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

                    Interface       Trusted    Rate limit (pps)
                    -----------------------    ----------------
FastEthernet0/5     no              unlimited
FastEthernet0/6     no              unlimited
GigabitEthernet0/1  yes             unlimited
GigabitEthernet0/2  yes             unlimited
```

Populated binding table from legitimate DHCP leases:

```
SW2#show ip dhcp snooping binding
MacAddress          IpAddress          Lease(sec)  Type           VLAN  Interface
------------------  -----------------  ----------  -------------  ----  -----------------
00:30:A3:54:3C:4E   192.168.20.14               0  dhcp-snooping    20  FastEthernet0/5
00:01:C9:78:6C:00   192.168.20.16               0  dhcp-snooping    20  FastEthernet0/6
Total number of bindings: 2
```

## Dynamic ARP Inspection

Trust model — every host port untrusted, both uplinks trusted:

```
SW2#show ip arp inspection interfaces

Interface  Trust State  Rate (pps)  Burst Interval
---------- -----------  ----------  --------------
Fa0/1      Untrusted    15          1
Fa0/2      Untrusted    15          1
Fa0/3      Untrusted    15          1
Fa0/4      Untrusted    15          1
Fa0/5      Untrusted    15          1
Fa0/6      Untrusted    15          1
Fa0/7      Untrusted    15          1
Fa0/8      Untrusted    15          1
Fa0/9      Untrusted    15          1
Fa0/10     Untrusted    15          1
Fa0/11     Untrusted    15          1
Fa0/12     Untrusted    15          1
Fa0/13     Untrusted    15          1
Fa0/14     Untrusted    15          1
Fa0/15     Untrusted    15          1
Fa0/16     Untrusted    15          1
Fa0/17     Untrusted    15          1
Fa0/18     Untrusted    15          1
Fa0/19     Untrusted    15          1
Fa0/20     Untrusted    15          1
Fa0/21     Untrusted    15          1
Fa0/22     Untrusted    15          1
Fa0/23     Untrusted    15          1
Fa0/24     Untrusted    15          1
Gig0/1     Trusted      15          1
Gig0/2     Trusted      15          1
```

DAI active per-VLAN, with full counters:

```
SW2#show ip arp inspection

Source Mac Validation      : Disabled
Destination Mac Validation : Disabled
IP Address Validation      : Disabled

Vlan  Configuration  Operation  ACL Match  Static ACL
----  -------------  ---------  ---------  ----------
  10  Enabled        Inactive
  20  Enabled        Active
  30  Enabled        Active

Vlan  ACL Logging  DHCP Logging  Probe Logging
----  -----------  ------------  -------------
  10  Deny         Deny          Off
  20  Deny         Deny          Off
  30  Deny         Deny          Off

Vlan  Forwarded  Dropped  DHCP Drops  ACL Drops
----  ---------  -------  ----------  ---------
  10          0        0           0          0
  20          5        0           0          0
  30          0        0           0          0

Vlan  DHCP Permits  ACL Permits  Probe Permits  Source MAC Failures
----  ------------  -----------  -------------  -------------------
  10             0            0              0                    0
  20             5            0              0                    0
  30             0            0              0                    0

Vlan  Dest MAC Failures  IP Validation Failures  Invalid Protocol Data
----  -----------------  ----------------------  ---------------------
  10                  0                       0                      0
  20                  0                       0                      0
  30                  0                       0                      0
```

Summary view, same counters:

```
SW2#show ip arp inspection statistics

Vlan  Forwarded  Dropped  DHCP Drops  ACL Drops
----  ---------  -------  ----------  ---------
  10          0        0           0          0
  20          5        0           0          0
  30          0        0           0          0

Vlan  DHCP Permits  ACL Permits  Source MAC Failures
----  ------------  -----------  -------------------
  10             0            0                    0
  20             5            0                    0
  30             0            0                    0

Vlan  Dest MAC Failures  IP Validation Failures
----  -----------------  ----------------------
  10                  0                      0
  20                  0                      0
  30                  0                      0
```

Reading it: VLAN 20 forwarded 5 ARP packets, all permitted via DHCP lease match (`DHCP Permits: 5`), zero failures. Inspection is active and validating against the snooping binding table.

## Known simulator limitation (documented honestly)

Packet Tracer implements DAI's trust model and counters but does not enforce the binding check: an unbound host's ARP is forwarded and the Dropped counter stays at zero (verified by pinging successfully with an empty binding table). On real IOS, GNS3 or EVE-NG, the identical configuration drops the frames (`DHCP Drops >= 1`). Readers on PT should treat the forwarded counters as evidence of active inspection and validate drop behavior on fuller tooling.
