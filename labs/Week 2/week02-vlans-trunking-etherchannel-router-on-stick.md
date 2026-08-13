# Week 2 Lab: VLANs, Trunking, STP, EtherChannel & Router-on-a-Stick

**Tool:** Cisco Packet Tracer
**Topology:** 2 PCs — Switch1 — (EtherChannel trunk) — Switch2 — Router
**Blueprint domains covered:** Network Access (VLANs, trunking, STP, EtherChannel), IP Connectivity (inter-VLAN routing)

## Objective

Segment two end devices into separate VLANs, connect two switches with a redundant link bundled into an EtherChannel, and configure a router to route between VLANs using router-on-a-stick — then troubleshoot the multi-hop path until full end-to-end connectivity is achieved.

## Topology

```
PC1 (VLAN 10 - Sales)  ---\
                            Switch1 ===(Po1: Fa0/1+Fa0/2, LACP, 802.1Q trunk)=== Switch2 --(Fa0/3, 802.1Q trunk)-- Router
PC2 (VLAN 20 - Engineering) --/
```

## VLAN Configuration (Switch1)

```
Switch1(config)# vlan 10
Switch1(config-vlan)# name SALES
Switch1(config-vlan)# exit
Switch1(config)# vlan 20
Switch1(config-vlan)# name ENGINEERING
Switch1(config-vlan)# exit

Switch1(config)# interface fastethernet0/3
Switch1(config-if)# switchport mode access
Switch1(config-if)# switchport access vlan 10
Switch1(config-if)# exit

Switch1(config)# interface fastethernet0/4
Switch1(config-if)# switchport mode access
Switch1(config-if)# switchport access vlan 20
Switch1(config-if)# exit
```

## Watching STP Work Live (Before EtherChannel)

Before bundling the two Switch1–Switch2 links, `show spanning-tree` showed classic STP loop prevention in action:

```
Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/2            Altn BLK 19        128.2    P2p
Fa0/1            Root FWD 19        128.1    P2p
```

With two physical cables between the same two switches, STP automatically elected one port as the Root Port (forwarding) and put the redundant port into Blocking state — preventing a Layer 2 loop without any manual configuration.

## EtherChannel Configuration

Bundled both physical links into a single logical Port-channel using LACP, on both switches:

```
Switch1(config)# interface range fastethernet0/1 - 2
Switch1(config-if-range)# channel-group 1 mode active
Switch1(config-if-range)# exit
Switch1(config)# interface port-channel 1
Switch1(config-if)# switchport mode trunk
Switch1(config-if)# exit
```

(Same commands mirrored on Switch2.)

**Result — `show etherchannel summary`:**
```
Group  Port-channel  Protocol    Ports
------+-------------+-----------+----------------------------------------------
1      Po1(SU)           LACP   Fa0/1(P) Fa0/2(P)
```

**Result — `show spanning-tree` after bundling:**
```
Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Po1              Root FWD 12        128.27   P2p
```

Both physical links now appear to STP as a single logical interface (Po1) — no port is blocked, and the network gets full combined bandwidth plus automatic failover if one cable fails.

## Router-on-a-Stick Configuration

```
Router(config)# interface gigabitethernet0/0/0.10
Router(config-subif)# encapsulation dot1Q 10
Router(config-subif)# ip address 192.168.10.1 255.255.255.0
Router(config-subif)# exit

Router(config)# interface gigabitethernet0/0/0.20
Router(config-subif)# encapsulation dot1Q 20
Router(config-subif)# ip address 192.168.20.1 255.255.255.0
Router(config-subif)# exit
```

PC gateways set accordingly:

| Device | IP | Subnet Mask | Default Gateway |
|---|---|---|---|
| PC1 (VLAN 10) | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 |
| PC2 (VLAN 20) | 192.168.20.10 | 255.255.255.0 | 192.168.20.1 |

## Troubleshooting: Inter-VLAN Ping Failure

Initial ping from PC1 to the VLAN 20 gateway (192.168.10.1 → 192.168.20.1) failed with 100% loss. Diagnosed methodically, one hop at a time:

### Step 1 — Confirm the topology itself

Assumed the router connected directly to Switch1. Used `show cdp neighbors` on Switch1 and found no router listed — only Switch2. Traced cables visually in Packet Tracer and confirmed the real topology: **PC → Switch1 → Switch2 → Router**, not PC → Switch1 → Router as originally assumed.

### Step 2 — Check the router-facing trunk on Switch2

`show cdp neighbors` on Switch2 confirmed the router connects via Fa0/3. `show interfaces trunk` showed only `Po1` trunking — **Fa0/3 was still a plain access port**, meaning tagged VLAN traffic couldn't reach the router at all.

**Fix:**
```
Switch2(config)# interface fastethernet0/3
Switch2(config-if)# switchport mode trunk
Switch2(config-if)# exit
```

### Step 3 — VLAN database gap between switches

After trunking Fa0/3, `show interfaces trunk` still showed only VLAN 1 as active (not 1,10,20). `show vlan brief` on Switch2 confirmed **VLANs 10 and 20 had never been created on Switch2** — trunk links can only pass traffic for VLANs the switch itself knows about, and VLAN creation is not automatically synced between switches without VTP.

**Fix:**
```
Switch2(config)# vlan 10
Switch2(config-vlan)# name SALES
Switch2(config-vlan)# exit
Switch2(config)# vlan 20
Switch2(config-vlan)# name ENGINEERING
Switch2(config-vlan)# exit
```

### Result

Ping from PC1 (192.168.10.10) to PC2 (192.168.20.10) succeeded. Full inter-VLAN routing confirmed working end-to-end:

```
PC1 (VLAN 10) → Switch1 → Po1 trunk → Switch2 → Fa0/3 trunk → Router (sub-interfaces route between VLANs) → back down the same path → PC2 (VLAN 20)
```

## Key Takeaways

- **STP requires zero manual configuration** to prevent loops — it runs by default and blocks redundant paths automatically, visible directly in `show spanning-tree`.
- **EtherChannel changes what STP sees**, not what physically exists — bundling links into one logical Port-channel removes the loop risk STP was blocking against, without removing the physical redundancy.
- **Trunk links must be configured consistently on every switch hop** in the path, not just the first one closest to the end devices. A single un-trunked port anywhere along the path breaks VLAN-tagged traffic.
- **VLANs must be created individually on every switch that needs to pass or terminate their traffic** — a trunk port can carry VLANs it's configured to allow, but only if the switch's VLAN database actually contains those VLANs. Without VTP, this isn't automatic.
- **CDP (`show cdp neighbors`) is a critical verification tool** — physical topology assumptions should always be confirmed against actual CDP output or a visual cable trace rather than assumed from memory.
