[README.md](https://github.com/user-attachments/files/34654148/README.md)
# IGMPv2 + PIM-SM with and without IGMP Snooping — Nokia SR OS + BNG Blaster

The same multicast traffic is delivered to two LANs: LAN-A has no snooping, and LAN-B has IGMP snooping. BNG Blaster measures, in milliseconds, how long traffic takes to arrive after a join and how long it takes to stop after a leave.

```
 src ──── R2 ─────────────────────────── R1 ──── SW1 ─┬─ LAN-A (VPLS 100, no snooping): rcv-a1, rcv-a2, h-linux-a
 10.10.0.10  \                          /             └─ LAN-B (VPLS 200, snooping):    rcv-b1, rcv-b2, h-linux-b
              └──── R3 ────── R4 ──────┘
                    RP 3.3.3.3
```

The shared tree runs through the RP (R2 → R3 → R4 → R1). The shortest-path tree, or SPT, uses the direct link (R2 → R1).

| Node | Role | Interfaces |
|---|---|---|
| R1 (1.1.1.1) | Receiver DR, IGMPv2 querier | to-r2 10.0.12.1/30 · to-r4 10.0.14.1/30 · to-lan-a 10.100.1.1/24 · to-lan-b 10.100.2.1/24 |
| R2 (2.2.2.2) | Source DR | to-src 10.10.0.1/24 · to-r1 10.0.12.2/30 · to-r3 10.0.23.1/30 |
| R3 (3.3.3.3) | Rendezvous Point | to-r2 10.0.23.2/30 · to-r4 10.0.34.1/30 |
| R4 (4.4.4.4) | Transit on the shared tree | to-r3 10.0.34.2/30 · to-r1 10.0.14.2/30 |
| SW1 | L2 switch | VPLS 100 (LAN-A) · VPLS 200 (LAN-B, fast-leave on 1/2/c1/1) |
| src | Multicast source | Sends 239.1.1.1 and 239.1.1.2 at 50 pps each |
| rcv-a1, rcv-b1 | Receivers | Join 239.1.1.1 (autostart) |
| rcv-a2, rcv-b2 | Receivers | Join 239.1.1.2 (autostart) |

All routers run OSPF area 0 and use a static RP, 3.3.3.3. The IGMP timers on R1 are: query interval 30 s, last member query interval 1 s, robust count 2. That gives a group membership interval of 70 s.

## Deploy

```bash
clab deploy -t Lab_igmpv2.clab.yaml
```

On every router, apply `configs/00_modo_mixed.txt`. Then paste `r2.cfg`, `r3.cfg`, `r4.cfg`, `r1.cfg` and `sw1.cfg` into their nodes, using classic CLI.

To start the source:

```bash
docker exec -d clab-igmpv2-src sh -c "rm -f /run/bngblaster.sock /run/lock/bngblaster_eth1.lock; bngblaster -C /config/src.json -S /run/bngblaster.sock -L /config/out/src.log -l info"
```

To start the receivers:

```bash
for n in rcv-a1 rcv-a2 rcv-b1 rcv-b2; do docker exec -d clab-igmpv2-$n sh -c "rm -f /run/bngblaster.sock /run/lock/bngblaster_eth1.lock; bngblaster -C /config/$n.json -S /run/bngblaster.sock -L /config/out/$n.log -l info -l igmp -P /config/out/$n.pcap"; done
```

## Verify

On R1, check the RPF interfaces. The source should resolve via `to-r2` (the SPT), and the RP via `to-r4` (the shared tree):

```
show router pim rpf 10.10.0.10
```

```
show router pim rpf 3.3.3.3
```

On R1, the (\*,G) should arrive on `to-r4` and the (S,G) on `to-r2`:

```
show router pim group
```

On SW1, VPLS 200 should map each group only to the port that requested it. VPLS 100 has no MFIB, so it floods everything:

```
show service id 200 mfib
```

## Tests

This shows per-group state for a receiver, including `packets`, `join-delay-ms` and `leave-delay-ms`:

```bash
docker exec clab-igmpv2-rcv-a1 bngblaster-cli /run/bngblaster.sock igmp-info session-id 1
```

This shows the multicast each receiver gets, whether it asked for it or not:

```bash
for n in rcv-a1 rcv-a2 rcv-b1 rcv-b2; do echo -n "$n: "; docker exec clab-igmpv2-$n bngblaster-cli /run/bngblaster.sock access-interfaces | grep rx-pps-multicast; done
```

**Test 1 — Flooding.** Expect about 100 pps on each LAN-A receiver, which gets both groups. Expect about 50 pps on each LAN-B receiver, which gets only its own group.

**Test 2 — Leave.** rcv-b1 has no fast-leave, so expect a `leave-delay-ms` of about 2000 ms. rcv-b2 has fast-leave, so expect tens of milliseconds.

```bash
docker exec clab-igmpv2-rcv-b1 bngblaster-cli /run/bngblaster.sock igmp-leave session-id 1 group 239.1.1.1
```

```bash
docker exec clab-igmpv2-rcv-b2 bngblaster-cli /run/bngblaster.sock igmp-leave session-id 1 group 239.1.1.2
```

**Test 3 — Leave while another member remains.** Join rcv-a2 and rcv-b2 to 239.1.1.1, then have rcv-a1 and rcv-b1 leave it. In LAN-A, rcv-a1 keeps receiving traffic, because R1 decides per interface. In LAN-B, rcv-b1 stops after about 2000 ms, because SW1 prunes per port.

```bash
docker exec clab-igmpv2-rcv-a2 bngblaster-cli /run/bngblaster.sock igmp-join session-id 1 group 239.1.1.1
```

```bash
docker exec clab-igmpv2-rcv-a1 bngblaster-cli /run/bngblaster.sock igmp-leave session-id 1 group 239.1.1.1
```

Then repeat the same two commands for rcv-b2 and rcv-b1.

| Test | LAN-A | LAN-B |
|---|---|---|
| 1. pps per receiver | | |
| 2. leave-delay-ms | | |
| 3. leave-delay-ms, another member remains | | |

## Notes

- BNG Blaster transmits through a kernel bypass, so a capture on its own `eth1` shows only received packets. To see what it sends, capture on the neighbor's port (for example `r2:1/1/c1/1`) or open its pcap in `bngblaster/out/`.
- BNG Blaster sends IGMPv2 Leaves to the group address. The standard specifies 224.0.0.2.
- SW1 summarizes Reports toward R1 on LAN-B, so the Last Reporter that R1 shows on `to-lan-b` may not be the host's IP.

**In short:** IGMP requests the traffic, PIM delivers it between routers, and snooping filters it on the switch.
