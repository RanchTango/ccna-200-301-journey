# Week 1 Lab: Cabling, Router Configuration & Fault Injection

**Tool:** Cisco Packet Tracer
**Topology:** 2 PCs — Switch (2960) — Router (1941)
**Blueprint domains covered:** Network Fundamentals (cabling, IP addressing), Network Access (switching basics)

## Objective

Build a small routed network from scratch, bring it to full end-to-end connectivity, then deliberately introduce faults at different OSI layers to practice bottom-up troubleshooting methodology.

## Topology Build

- Connected PC1 and PC2 to the switch using **straight-through** cabling (different device types)
- Connected the switch to the router using **straight-through** cabling
- Accessed the router via CLI and brought the interface online

## Router Configuration

Router interfaces are administratively shut down by default. Brought the LAN-facing interface online and assigned an IP:

```
Router> enable
Router# configure terminal
Router(config)# hostname R1
R1(config)# interface gigabitethernet0/0
R1(config-if)# no shutdown
R1(config-if)# ip address 192.168.1.1 255.255.255.0
R1(config-if)# end
R1# show ip interface brief
```

Result: `GigabitEthernet0/0` showed `Status: up, Protocol: up` with IP `192.168.1.1` assigned.

## End Device Configuration

| Device | IP Address | Subnet Mask | Default Gateway |
|---|---|---|---|
| PC1 | 192.168.1.10 | 255.255.255.0 | 192.168.1.1 |
| PC2 | 192.168.1.11 | 255.255.255.0 | 192.168.1.1 |

## Baseline Connectivity Test

- `ping 192.168.1.11` (PC1 → PC2): **4/4 success**
- `ping 192.168.1.1` (PC1 → Router): **4/4 success**

Confirmed full Layer 2 (switching) and Layer 3 (routing/addressing) connectivity across the topology.

## Fault Injection & Troubleshooting

Introduced three separate faults, one at a time, to practice isolating problems by OSI layer.

### Fault 1 — Physical cable disconnection (PC1 → Switch)

- **Symptom:** `ping 192.168.1.1` from PC1 timed out
- **Diagnostic approach:** tested connectivity from PC2 first to determine if the issue was network-wide or isolated to PC1 — confirmed PC2 was unaffected, narrowing the problem to PC1's local connection
- **Root cause:** physical cable disconnected between PC1 and the switch
- **OSI Layer:** 1 (Physical) — nothing above this layer can function without a physical link
- **Resolution:** reconnected the cable; verified with successful pings to both the gateway and PC2

### Fault 2 — Router interface administratively down

- **Symptom:** local LAN pings succeeded, but external connectivity (e.g., browsing to google.com) failed
- **Diagnostic approach:** the fact that local traffic worked but external traffic didn't pointed away from the LAN itself and toward the gateway/router
- **Root cause:** `shutdown` issued on the router's LAN interface, disabling it
- **OSI Layer:** 1/2 (Physical/Data Link) — the interface itself was down, so no traffic could pass through the router at all
- **Resolution:** ran `no shutdown` on the interface; verified with `show ip interface brief` (Status/Protocol both returned to `up`)

### Fault 3 — Incorrect subnet mask on PC2

- **Symptom:** PC2 specifically unreachable, even after Faults 1 and 2 were resolved
- **Diagnostic approach:** since the problem was isolated to a single device (not physical, not the gateway), checked IP configuration directly rather than re-checking cabling
- **Root cause:** PC2's subnet mask was set to `255.255.255.192` instead of `255.255.255.0`, causing it to miscalculate which addresses were on its local subnet
- **OSI Layer:** 3 (Network) — a logical addressing problem, not a physical or data link issue
- **Resolution:** corrected the subnet mask to `255.255.255.0`; verified with successful ping

## Key Takeaway

Each fault was diagnosed by moving methodically up the OSI stack — ruling out Physical Layer issues first, then Data Link, then Network — rather than guessing randomly. This bottom-up approach isolates root causes efficiently and mirrors real-world troubleshooting expectations for network technicians.
