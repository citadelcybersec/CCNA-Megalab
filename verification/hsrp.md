HSRP — First-Hop Redundancy Verification
Design intent

Per VLAN, HSRP Active and STP root are assigned to the same switch: MLSW1 is Active/root for VLANs 10 and 30, MLSW2 for VLAN 20. Endpoints use the virtual gateway (.5), never a physical switch.

Related screenshots: 03-hsrp-standby-stp.png, 04-hsrp-failover.png
Baseline roles (steady state)

MLSW1 — Active for Vl10 (pri 150) and Vl30, Standby for Vl20:
MLSW1#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   150 P Active   local           192.168.10.4    192.168.10.5
Vl20        20   100 P Standby  192.168.20.4    local           192.168.20.5
Vl30        30   150 P Active   local           192.168.30.4    192.168.30.5

MLSW2 — Active for Vl20 (pri 200), Standby for Vl10 and Vl30:
MLSW2#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   100 P Standby  192.168.10.3    local           192.168.10.5
Vl20        20   200 P Active   local           192.168.20.3    192.168.20.5
Vl30        30   100 P Standby  192.168.30.3    local           192.168.30.5
Root alignment

MLSW1 is root for VLAN 10 (priority 4096 + sys-id-ext 10 = 4106):
MLSW1#show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol ieee
  Root ID    Priority    4106
             Address     000A.F30E.697B
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    4106  (priority 4096 sys-id-ext 10)
             Address     000A.F30E.697B
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi1/5            Desg FWD 4         128.5    P2p

MLSW2 is root for VLAN 20 (priority 4096 + sys-id-ext 20 = 4116):
MLSW2#show spanning-tree vlan 20

VLAN0020
  Spanning tree enabled protocol ieee
  Root ID    Priority    4116
             Address     0002.1773.06EC
             This bridge is the root
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    4116  (priority 4096 sys-id-ext 20)
             Address     0002.1773.06EC
             Hello Time   2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi1/6            Desg FWD 4         128.6    P2p
Failure test: failover

Shutting down MLSW1's VLAN 10 SVI:
MLSW1#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
MLSW1(config)#int vlan 10
MLSW1(config-if)#shutdown

MLSW1(config-if)#
%LINK-5-CHANGED: Interface Vlan10, changed state to administratively down

%LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan10, changed state to down

MLSW2 takes the Active role immediately (console log captured on MLSW2):
MLSW2#
%HSRP-6-STATECHANGE: Vlan10 Grp 10 state Standby -> Active

Post-takeaway state on MLSW2. Standby shows unknown because the peer SVI is administratively down — expected during this test:
MLSW2#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   100 P Active   local           unknown         192.168.10.5
Vl20        20   200 P Active   local           192.168.20.3    192.168.20.5
Vl30        30   100 P Standby  192.168.30.3    local           192.168.30.5
Failure test: failback and preemption

Restoring the SVI on MLSW1 — priority 150 with standby preempt, it reclaims Active:
MLSW1(config-if)#no shutdown

MLSW1(config-if)#
%LINK-3-UPDOWN: Interface Vlan10, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface Vlan10, changed state to up

MLSW1(config-if)#
 %HSRP-6-STATECHANGE: Vlan10 Grp 10 state Standby -> Active

(Note: the %LINK-3-UPDOWN ... changed state to down line paired with line protocol ... up is how Packet Tracer reports the SVI coming back; the meaningful lines are the protocol-up and the HSRP state change.)

On MLSW2, group 10 walks back down through Speak to Standby as the preempted peer reclaims the role:
MLSW2#
%HSRP-6-STATECHANGE: Vlan10 Grp 10 state Speak -> Standby

Final state on MLSW2 — group 10 returned cleanly to Standby, MLSW2 still Active for its designed group 20:
MLSW2#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   100 P Standby  192.168.10.3    local           192.168.10.5
Vl20        20   200 P Active   local           192.168.20.3    192.168.20.5
Vl30        30   100 P Standby  192.168.30.3    local           192.168.30.5

And on MLSW1 — the designed role assignment fully restored after the test:
MLSW1#show standby brief
                     P indicates configured to preempt.
                     |
Interface   Grp  Pri P State    Active          Standby         Virtual IP
Vl10        10   150 P Active   local           192.168.10.4    192.168.10.5
Vl20        20   100 P Standby  192.168.20.4    local           192.168.20.5
Vl30        30   150 P Active   local           192.168.30.4    192.168.30.5
What the test demonstrated

    Failover: loss of the Active switch's SVI promoted the Standby within one hello interval, with the virtual gateway .5 preserved — clients keep their default gateway.
    Failback with preemption: restoration of the higher-priority switch reclaimed Active (Standby -> Active on MLSW1) and the peer stepped down gracefully (Speak -> Standby on MLSW2) — no manual intervention, no split brain.
    Role alignment held throughout: MLSW2 remained Active for Vl20 (its designed group) the entire time; the test affected only group 10.
