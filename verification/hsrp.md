# HSRP — First-Hop Redundancy Verification

## Design intent

Per VLAN, HSRP Active and STP root are assigned to the same switch:
MLSW1 is Active/root for VLANs 10 and 30, MLSW2 for VLAN 20. Endpoints
use the virtual gateway (.5), never a physical switch.

Related screenshots: `03-hsrp-standby-stp.png`, `04-hsrp-failover.png`

## Baseline roles (steady state)

MLSW1 — Active for Vl10 (pri 150) and Vl30, Standby for Vl20:

```
MLSW1#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   150 P Active   local           192.168.10.4    192.168.10.5
Vl20        20   100 P Standby  192.168.20.4    local           192.168.20.5
Vl30        30   150 P Active   local           192.168.30.4    192.168.30.5
```

MLSW2 — Active for Vl20 (pri 200), Standby for Vl10 and Vl30:

```
MLSW2#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   100 P Standby  192.168.10.3    local           192.168.10.5
Vl20        20   200 P Active   local           192.168.20.3    192.168.20.5
Vl30        30   100 P Standby  192.168.30.3    local           192.168.30.5
```

## Root alignment

MLSW1 is root for VLAN 10 (priority 4096 + 10 sys-id-ext = 4106):

```
MLSW1#show spanning-tree vlan 10
VLAN0010
  Spanning tree enabled protocol ieee
  Root ID    Priority    4106
             Address     000A.F30E.697B
             This bridge is the root
...
  Interface        Role Sts Cost      Prio.Nbr Type
----------------   ---- --- --------- -------- --------------------------
Gi1/5              Desg FWD 4         128.5    P2p
```

MLSW2 is root for VLAN 20 (priority 4096 + 20 = 4116):

```
MLSW2#show spanning-tree vlan 20
VLAN0020
  Spanning tree enabled protocol ieee
  Root ID    Priority    4116
             Address     0002.1773.06EC
             This bridge is the root
...
  Interface        Role Sts Cost      Prio.Nbr Type
----------------   ---- --- --------- -------- --------------------------
Gi1/6              Desg FWD 4         128.6    P2p
```

## Failure test: failover

Shutting down MLSW1's VLAN 10 SVI:

```
MLSW1(config)#int vlan 10
MLSW1(config-if)#shutdown
%LINK-5-CHANGED: Interface Vlan10, changed state to administratively down
%LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan10, changed state to down
```

MLSW2 takes the Active role immediately (console log captured from MLSW2):

```
MLSW2#
%HSRP-6-STATECHANGE: Vlan10 Grp 10 state Standby -> Active
```

Post-takeaway state. Standby shows `unknown` because the peer SVI is
administratively down — expected during this test:

```
MLSW2#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   100 P Active   local           unknown         192.168.10.5
Vl20        20   200 P Active   local           192.168.20.3    192.168.20.5
Vl30        30   100 P Standby  192.168.30.3    local           192.168.30.5
```

## Failure test: failback and preemption

Restoring the SVI — MLSW1, priority 150 with `standby preempt`,
reclaims Active; note the state change fires on MLSW1's console:

```
MLSW1(config-if)#no shutdown
%LINK-3-UPDOWN: Interface Vlan10, changed state to down
%LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan10, changed state to up
 %HSRP-6-STATECHANGE: Vlan10 Grp 10 state Standby -> Active
```

Final role table on MLSW1 — Vl10 restored to Active with preempt.
Vl20 shows `Speak` at this instant: group 20 was still churning from
the convergence disturbance (see the note below), which is why the
raw output is reproduced verbatim rather than tidied:

```
MLSW1#do show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   150 P Active   local           192.168.10.4    192.168.10.5
Vl20        20   100 P Speak    192.168.20.3    unknown         192.168.20.5
Vl30        30   150 P Active   local           192.168.30.4    192.168.30.5
```

### Observed side effect

During the Vl10 recovery window, group 20 showed repeated
`Speak -> Standby -> Active` transitions on MLSW1 — hello loss on the
inter-switch path during convergence churn, amplified by Packet
Tracer's coarse timers. State settled to the designed roles shortly
afterward.
