# EVPN Layer 2 and Layer 3 Configuration

Hello guys, how are you doing? I hope everything is going well!

Today, we'll dive into a fun topic that I fell in love with on day one: EVPN!

EVPN is essentially the natural evolution of L2VPNs. I say L2VPNs in general because EVPN has several flavors that I won't fully explore here. But beyond Layer 2, we can also integrate L2 EVPNs directly into VRFs, effectively creating L3 EVPNs!

Here is the topology we'll be using today:
<img width="1507" height="1138" alt="image" src="https://github.com/user-attachments/assets/516631b6-f67d-442b-a514-99307dff7a3a" />

The scenario is as follows: our customer wants Layer 2 and Layer 3 connectivity between their sites, plus internet access.
First, we'll configure a standard Layer 2 EVPN. Then, to provide internet access and routing, we'll stitch the EVPN into a VRF. All right, let's get to it.

First things first, here is the reference table for our EVPN configuration:
| CE Router | CE's IP Address | PE | PE-CE Interface | LACP  | VLAN ID | Default Gateway |
| ----------- | ----------------- | ----------- | ------------------- | ----- | ------- | ---------------- |
| CE12-1      | 10.16.12.12/24   | R1          | ae1.1200            | Active | 1200    | 10.16.12.254/24 |
| CE12-1      | 10.16.12.12/24   | R2          | ae1.1200            | Active | 1200    | 10.16.12.254/24 |
| CE34-1      | 10.16.34.34/24   | R3          | ae1.3400            | Active | 3400    | 10.16.34.254/24 |
| CE34-1      | 10.16.34.34/24   | R4          | ae1.3400            | Active | 3400    | 10.16.34.254/24 |

Let's enable the evpn family in our BGP mesh:: 
RR:
```
set protocols bgp group iBGP-AS65020-West family evpn signaling nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-West family evpn signaling no-install
set protocols bgp group iBGP-AS65020-East family evpn signaling nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-East family evpn signaling no-install
```
PEs:
```
set protocols bgp group iBGP-AS65020-East family evpn signaling
or
set protocols bgp group iBGP-AS65020-West family evpn signaling
```
Ok, with the EVPN family running across the backbone, we can configure our EVPN Instances (EVI)!

Both sites belong to the same customer, and both are multi-homed. Unlike VPLS, EVPN allows us to implement true active-active multi-homing using ESI-LAGs!
How does this work? Basically, we configure LACP on both PEs using the exact same system-ID. From the customer's perspective, the two PEs look like a single logical switch, and outbound traffic is naturally load-balanced across the links. Return traffic destined for the CE is load-balanced using the EVPN Aliasing label! To prevent forwarding loops, a Designated Forwarder (DF) is elected to handle BUM (Broadcast, Unknown Unicast, Multicast) traffic. It’s an incredibly elegant design, I must confess.

Another key detail in our topology is the VLAN mapping. To allow communication between the two sites, we'll perform VLAN normalization, translating the local VLAN tags to a common VLAN of 1234 in the core.

Let's start with the configuration on R1 and R2:
R1:
```
set interfaces ge-0/0/6 description to-CE12-1
set interfaces ge-0/0/6 gigether-options 802.3ad ae1

set interfaces ae1 description to-CE12-1
set interfaces ae1 flexible-vlan-tagging
set interfaces ae1 encapsulation flexible-ethernet-services
set interfaces ae1 esi 00:00:00:00:00:00:00:00:00:01
set interfaces ae1 esi all-active
set interfaces ae1 aggregated-ether-options lacp active
set interfaces ae1 aggregated-ether-options lacp system-id 00:00:00:00:00:01
set interfaces ae1 unit 1200 encapsulation vlan-bridge
set interfaces ae1 unit 1200 vlan-id 1200

set routing-instances EVPN-CE12 instance-type evpn
set routing-instances EVPN-CE12 protocols evpn encapsulation mpls
set routing-instances EVPN-CE12 description EVPN-CE12
set routing-instances EVPN-CE12 vlan-id 1234
set routing-instances EVPN-CE12 interface ae1.1200
set routing-instances EVPN-CE12 route-distinguisher 10.0.0.1:1234
set routing-instances EVPN-CE12 vrf-target target:65020:1234
```
R2:
```
set interfaces ge-0/0/6 description to-CE12-1
set interfaces ge-0/0/6 gigether-options 802.3ad ae1

set interfaces ae1 description to-CE12-1
set interfaces ae1 flexible-vlan-tagging
set interfaces ae1 encapsulation flexible-ethernet-services
set interfaces ae1 esi 00:00:00:00:00:00:00:00:00:01
set interfaces ae1 esi all-active
set interfaces ae1 aggregated-ether-options lacp active
set interfaces ae1 aggregated-ether-options lacp system-id 00:00:00:00:00:01
set interfaces ae1 unit 1200 encapsulation vlan-bridge
set interfaces ae1 unit 1200 vlan-id 1200

set routing-instances EVPN-CE12 instance-type evpn
set routing-instances EVPN-CE12 protocols evpn encapsulation mpls
set routing-instances EVPN-CE12 description EVPN-CE12
set routing-instances EVPN-CE12 vlan-id 1234
set routing-instances EVPN-CE12 interface ae1.1200
set routing-instances EVPN-CE12 route-distinguisher 10.0.0.2:1234
set routing-instances EVPN-CE12 vrf-target target:65020:1234
```
First, we bind the physical interface to the LAG, and on the LAG, we define the Ethernet Segment Identifier (ESI) and its operational mode (which will be all-active). The LACP system-id must be identical on both PEs.

In the routing instance, we create an EVI and set the encapsulation to MPLS (though strictly speaking, you don't need to specify this, as MPLS is the default in Junos). Just like a L3VPN, the EVI requires an RD and RT! Borrowing these BGP components is exactly what makes EVPN so fantastic and scalable. Notice the vlan-id 1234 knob, this handles the VLAN normalization.

Now, let's configure the other site so we can run some tests. We'll apply the exact same logic:
R3:
```
set interfaces ge-0/0/9 description to-CE34-1
set interfaces ge-0/0/9 gigether-options 802.3ad ae1

set interfaces ae1 description to-CE34-1
set interfaces ae1 flexible-vlan-tagging
set interfaces ae1 encapsulation flexible-ethernet-services
set interfaces ae1 esi 00:00:00:00:00:00:00:00:00:02
set interfaces ae1 esi all-active
set interfaces ae1 aggregated-ether-options lacp active
set interfaces ae1 aggregated-ether-options lacp system-id 00:00:00:00:00:02
set interfaces ae1 unit 3400 encapsulation vlan-bridge
set interfaces ae1 unit 3400 vlan-id 3400

set routing-instances EVPN-CE34 instance-type evpn
set routing-instances EVPN-CE34 protocols evpn encapsulation mpls
set routing-instances EVPN-CE34 description EVPN-CE34
set routing-instances EVPN-CE34 vlan-id 1234
set routing-instances EVPN-CE34 interface ae1.3400
set routing-instances EVPN-CE34 route-distinguisher 10.0.0.3:1234
set routing-instances EVPN-CE34 vrf-target target:65020:1234
```
R4:
```
set interfaces ge-0/0/6 description to-CE34-1
set interfaces ge-0/0/6 gigether-options 802.3ad ae1

set interfaces ae1 description to-CE34-1
set interfaces ae1 flexible-vlan-tagging
set interfaces ae1 encapsulation flexible-ethernet-services
set interfaces ae1 esi 00:00:00:00:00:00:00:00:00:02
set interfaces ae1 esi all-active
set interfaces ae1 aggregated-ether-options lacp active
set interfaces ae1 aggregated-ether-options lacp system-id 00:00:00:00:00:02
set interfaces ae1 unit 3400 encapsulation vlan-bridge
set interfaces ae1 unit 3400 vlan-id 3400

set routing-instances EVPN-CE34 instance-type evpn
set routing-instances EVPN-CE34 protocols evpn encapsulation mpls
set routing-instances EVPN-CE34 description EVPN-CE34
set routing-instances EVPN-CE34 vlan-id 1234
set routing-instances EVPN-CE34 interface ae1.3400
set routing-instances EVPN-CE34 route-distinguisher 10.0.0.4:1234
set routing-instances EVPN-CE34 vrf-target target:65020:1234
```
With this committed, we are now learning MAC addresses from the PE-CE interfaces, but we still don't have ping connectivity between the sites. Do you know why? Because their IP addresses don't belong to the same subnet! We only have pure Layer 2 connectivity right now. Let's look at the LLDP neighbors...
```
[admin@CE12-1] > ip nei pr
Columns: INTERFACE, ADDRESS, MAC-ADDRESS, IDENTITY, VERSION, BOARD
#  INTERFACE  ADDRESS      MAC-ADDRESS        IDENTITY  VERSION                            BOARD
0  vlan1200   10.12.34.34  50:A8:C0:00:2A:00  CE34-1    7.8 (stable) Feb/24/2023 09:03:00  CHR
----------
[admin@CE34-1] > ip nei pr
Columns: INTERFACE, ADDRESS, MAC-ADDRESS, IDENTITY, VERSION, BOARD
#  INTERFACE  ADDRESS      MAC-ADDRESS        IDENTITY  VERSION                            BOARD
0  vlan3400   10.16.12.12  50:EA:05:00:1C:00  CE12-1    7.8 (stable) Feb/24/2023 09:03:00  CHR
```
Both CEs show LLDP neighbor adjacency, confirming L2 is up, but there's no Layer 3 routing yet.

Let's check the EVPN routing table to see what routes we have acquired so far:
```
root@R1> show route table EVPN-CE12.evpn.0

EVPN-CE12.evpn.0: 15 destinations, 15 routes (15 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.0.0.1:1234::01::0/192 AD/EVI
                   *[EVPN/170] 01:02:34
                       Indirect
1:10.0.0.2:0::01::FFFF:FFFF/192 AD/ESI
                   *[BGP/170] 01:02:23, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 0
1:10.0.0.2:1234::01::0/192 AD/EVI
                   *[BGP/170] 01:02:34, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 22
1:10.0.0.3:0::02::FFFF:FFFF/192 AD/ESI
                   *[BGP/170] 01:02:22, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 35
                       to 10.200.0.3 via ge-0/0/2.0, Push 36
1:10.0.0.3:1234::02::0/192 AD/EVI
                   *[BGP/170] 01:02:32, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 22, Push 35(top)
                       to 10.200.0.3 via ge-0/0/2.0, Push 22, Push 36(top)
1:10.0.0.4:0::02::FFFF:FFFF/192 AD/ESI
                   *[BGP/170] 01:02:23, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 0
1:10.0.0.4:1234::02::0/192 AD/EVI
                   *[BGP/170] 01:02:35, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 22
2:10.0.0.1:1234::1234::50:ea:05:00:1c:00/304 MAC/IP
                   *[EVPN/170] 01:01:53
                       Indirect
2:10.0.0.2:1234::1234::50:ea:05:00:1c:00/304 MAC/IP
                   *[BGP/170] 01:01:53, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 22
2:10.0.0.3:1234::1234::50:a8:c0:00:2a:00/304 MAC/IP
                   *[BGP/170] 01:01:51, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 22, Push 35(top)
                       to 10.200.0.3 via ge-0/0/2.0, Push 22, Push 36(top)
2:10.0.0.4:1234::1234::50:a8:c0:00:2a:00/304 MAC/IP
                   *[BGP/170] 01:01:47, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 22
3:10.0.0.1:1234::1234::10.0.0.1/248 IM
                   *[EVPN/170] 4w6d 22:34:53
                       Indirect
3:10.0.0.2:1234::1234::10.0.0.2/248 IM
                   *[BGP/170] 00:05:45, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 0
3:10.0.0.3:1234::1234::10.0.0.3/248 IM
                   *[BGP/170] 00:05:45, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 35
                       to 10.200.0.3 via ge-0/0/2.0, Push 36
3:10.0.0.4:1234::1234::10.0.0.4/248 IM
                   *[BGP/170] 00:05:45, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 0
```
Here we have EVPN Route Types 1, 2, and 3.

**EVPN Type 1 Routes**: We actually have two sub-types here, haha.
**Ethernet AD Per-ESI**: This route is generated for the entire ESI. If the PE is connected to an ESI that spans multiple EVIs, it generates a single route containing the Route Targets (RTs) of all EVIs active on that ESI. (This is a key difference from the Per-EVI route, which uses the MPLS label field in the NLRI). This route also carries the ESI number and a special community containing the ESI Label.
This route serves two vital functions:

Mass Withdrawal: If the PE-CE physical link fails, the PE withdraws this single Type 1 route. Remote PEs immediately know to converge traffic over to the surviving PE in the ESI. Imagine having 500 MACs learned across dozens of EVIs on that interface. Instead of waiting to withdraw 500 individual MAC/IP routes, the single AD Per-ESI withdrawal instantly shifts the traffic. Make sense?

Loop Prevention (Split Horizon): This is achieved through the extended community attached to the route. It contains a bit indicating if the ESI is single-active or active-active, and crucially, the special ESI Label. Now follow my train of thought here: if CE12-1 sends a broadcast frame to R2, R2 will flood this traffic across the core to R1 (because R1 might be the DF for other sites). If R1 just blindly forwarded it back down to CE12-1, we'd have a loop. To solve this, when R2 floods BUM traffic to another PE in the same ESI, it pushes this special ESI label at the bottom of the stack. When R1 receives the frame and pops the label, it recognizes the ESI Label ("I AM BUM TRAFFIC FROM ESI-1, DO NOT SEND ME BACK TO ESI-1"). R1 will forward it to other local interfaces but drop it for the originating ESI. This label is also known as the Split Horizon Label.

**Ethernet AD Per-EVI** This route is generated per EVI active on the ESI. It has two main jobs: enabling load-balancing in all-active mode (Aliasing), and enabling fast backup paths in single-active mode.

Aliasing: Let's follow the thought process again. CE12-1 sends a unicast frame hashing to R1. R1 learns the source MAC and advertises it via BGP. But what if a remote PE (say, R3) needs to send traffic back to that MAC before R2 has naturally learned it? Since R3 knows R1 and R2 share the ESI, it wants to load-balance across both PEs. The Per-EVI route advertises an Aliasing Label. Remote PEs use this label to send traffic to R2, and R2 forwards it down to the ESI even if the specific MAC isn't populated in its local table yet!

Backup Paths: In single-active setups, remote PEs use this route to install the standby PE as a backup path in the FIB. When the primary PE's AD Per-ESI route is withdrawn, the backup path instantly becomes active.

You get the idea, right? Both of these Type 1 routes work together to advertise the presence of an ESI, distribute the topology, and dictate how traffic should be forwarded efficiently.

In the output, you can see the AD Per-ESI formatted as ```ROUTE-TYPE:RD::ESI```. Notice the esi-label community and the RT. Right below it, you see the AD Per-EVI formatted as ```ROUTE-TYPE:RD::ESI::0/192```, carrying the Aliasing label.
```
root@R1> show route table EVPN-CE12.evpn.0 match-prefix 1:* detail
....
1:10.0.0.2:0::01::FFFF:FFFF/192 AD/ESI (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.0.0.2:0
                Next hop type: Indirect, Next hop index: 0
                Address: 0x1cce3994
                Next-hop reference count: 8
                Kernel Table Id: 0
                Source: 10.0.0.0
                Protocol next hop: 10.0.0.2
                Indirect next hop: 0x2 no-forward INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Secondary Active Int Ext>
                Local AS: 65020 Peer AS: 65020
                Age: 2:06:34    Metric2: 5
                Validation State: unverified
                Task: BGP_65020.10.0.0.0
                Announcement bits (2): 0-EVPN-CE12-evpn 1-EVPN-CE12-evpn
                AS path: I  (Originator)
                Cluster list:  0.0.0.2
                Originator ID: 10.0.0.2
                Communities: target:65020:1234 esi-label:0x0:all-active (label 23)
                Import Accepted
                Localpref: 100
                Router ID: 10.0.0.0
                Primary Routing Table: bgp.evpn.0
                Thread: junos-main

1:10.0.0.2:1234::01::0/192 AD/EVI (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.0.0.2:1234
                Next hop type: Indirect, Next hop index: 0
                Address: 0x1cceba14
                Next-hop reference count: 4
                Kernel Table Id: 0
                Source: 10.0.0.0
                Protocol next hop: 10.0.0.2
                Label operation: Push 22
                Label TTL action: prop-ttl
                Load balance label: Label 22: None;
                Indirect next hop: 0x2 no-forward INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Secondary Active Int Ext>
                Local AS: 65020 Peer AS: 65020
                Age: 2:06:45    Metric2: 5
                Validation State: unverified
                Task: BGP_65020.10.0.0.0
                Announcement bits (1): 0-EVPN-CE12-evpn
                AS path: I  (Originator)
                Cluster list:  0.0.0.2
                Originator ID: 10.0.0.2
                Communities: target:65020:1234
                Import Accepted
                Route Label: 22
                Localpref: 100
                Router ID: 10.0.0.0
                Primary Routing Table: bgp.evpn.0
                Thread: junos-main
```
**EVPN Type 2 Routes** (MAC/IP Advertisement Route): These routes are responsible for advertising MAC addresses. They can also advertise MAC+IP bindings simultaneously via ARP snooping! These routes contain the MPLS label remote PEs must use to reach that MAC, the VLAN/Bridge-Domain the MAC belongs to, and the ESI where it was learned.
The structure is: ```ROUTE-TYPE:RD::VLAN::MAC-ADDRESS```.
```
root@R1> show route table EVPN-CE12.evpn.0 match-prefix 2:* detail
....
2:10.0.0.2:1234::1234::50:ea:05:00:1c:00/304 MAC/IP (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.0.0.2:1234
                Next hop type: Indirect, Next hop index: 0
                Address: 0x1cceba14
                Next-hop reference count: 4
                Kernel Table Id: 0
                Source: 10.0.0.0
                Protocol next hop: 10.0.0.2
                Label operation: Push 22
                Label TTL action: prop-ttl
                Load balance label: Label 22: None;
                Indirect next hop: 0x2 no-forward INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Secondary Active Int Ext>
                Local AS: 65020 Peer AS: 65020
                Age: 1:07:22    Metric2: 5
                Validation State: unverified
                Task: BGP_65020.10.0.0.0
                Announcement bits (1): 0-EVPN-CE12-evpn
                AS path: I  (Originator)
                Cluster list:  0.0.0.2
                Originator ID: 10.0.0.2
                Communities: target:65020:1234
                Import Accepted
                Route Label: 22
                ESI: 00:00:00:00:00:00:00:00:00:01
                Localpref: 100
                Router ID: 10.0.0.0
                Primary Routing Table: bgp.evpn.0
                Thread: junos-main
....
```
**EVPN Type 3 Routes** (Inclusive Multicast Ethernet Tag Route): This route builds an inclusive forwarding tree used for BUM traffic (Broadcast, Unknown Unicast, and Multicast). It’s heavily inspired by MVPNs! Hahaha, as Lavoisier said: nothing is created, everything is transformed.

In the output below, you see many of the same characteristics as a Type 2 route, but it features the router-ID of the remote PE instead of a MAC/IP. Crucially, the route details display the PMSI Tunnel attribute and the MPLS label that will be used to forward the BUM traffic across the core.
```
root@R1> show route table EVPN-CE12.evpn.0 match-prefix 3:* detail
....
3:10.0.0.2:1234::1234::10.0.0.2/248 IM (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.0.0.2:1234
                PMSI: Flags 0x0: Label 32: Type INGRESS-REPLICATION 10.0.0.2
                Next hop type: Indirect, Next hop index: 0
                Address: 0x1cce3994
                Next-hop reference count: 8
                Kernel Table Id: 0
                Source: 10.0.0.0
                Protocol next hop: 10.0.0.2
                Indirect next hop: 0x2 no-forward INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Secondary Active Int Ext>
                Local AS: 65020 Peer AS: 65020
                Age: 21:41      Metric2: 5
                Validation State: unverified
                Task: BGP_65020.10.0.0.0
                Announcement bits (1): 1-EVPN-CE12-evpn
                AS path: I  (Originator)
                Cluster list:  0.0.0.2
                Originator ID: 10.0.0.2
                Communities: target:65020:1234
                Import Accepted
                Route Label: 32
                Localpref: 100
                Router ID: 10.0.0.0
                Primary Routing Table: bgp.evpn.0
                Thread: junos-main
....
```

And, we have a hidden EVPN route working behind the scenes in our topology: the Type 4 route!!!

**EVPN Type 4 Routes** (Ethernet Segment Route): This route allows PEs to discover other PEs connected to the exact same ESI. We mentioned earlier that we need to prevent BUM traffic loops caused by local CEs. The Type 4 route handles the DF (Designated Forwarder) election to solve this.

To prevent duplicate flooding on an All-Active segment, only one PE is allowed to forward BUM traffic out to the CE. This election occurs per EVI on the ESI. The PEs use the Type 4 routes to discover each other, and then run a deterministic mathematical algorithm (defined in RFC 7432) to elect the DF. Because both PEs run the exact same math, they seamlessly agree on who takes the forwarding role.

The route format is ```ROUTE-TYPE:RD::ESI:PE```. Ultimately, R1 receives this route from R2 and says, "Ah, R2 is also connected to ESI 01. Let's run the DF election." For the rest of the PEs in the network, they see the MAC/IP routes pointing to the ESI next-hop and know traffic can go to either R1 or R2.
```
root@R1> show route table bgp.evpn.0 match-prefix 4:* detail

bgp.evpn.0: 22 destinations, 22 routes (22 active, 0 holddown, 0 hidden)
4:10.0.0.1:0::01:10.0.0.1/296 ES (1 entry, 1 announced)
        *EVPN   Preference: 170
                Next hop type: Indirect, Next hop index: 0
                Address: 0x8099f14
                Next-hop reference count: 17
                Kernel Table Id: 0
                Protocol next hop: 10.0.0.1
                Indirect next hop: 0x0 - INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Secondary Active Int Ext>
                Age: 1:42:55
                Validation State: unverified
                Task: __default_evpn__-evpn
                Announcement bits (1): 1-BGP_RT_Background
                AS path: I
                Communities: es-import-target:0-0-0-0-0-0
                Primary Routing Table: __default_evpn__.evpn.0
                Thread: junos-main

4:10.0.0.2:0::01:10.0.0.2/296 ES (1 entry, 0 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.0.0.2:0
                Next hop type: Indirect, Next hop index: 0
                Address: 0x1cce3994
                Next-hop reference count: 8
                Kernel Table Id: 0
                Source: 10.0.0.0
                Protocol next hop: 10.0.0.2
                Indirect next hop: 0x2 no-forward INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Active Int Ext>
                Local AS: 65020 Peer AS: 65020
                Age: 1:42:55    Metric2: 5
                Validation State: unverified
                Task: BGP_65020.10.0.0.0
                AS path: I  (Originator)
                Cluster list:  0.0.0.2
                Originator ID: 10.0.0.2
                Communities: es-import-target:0-0-0-0-0-0
                Import Accepted
                Localpref: 100
                Router ID: 10.0.0.0
                Secondary Tables: __default_evpn__.evpn.0
                Thread: junos-main
```


Now, checking the EVI status, we can see a beautiful summary of everything: the VLAN, the MAC database stats, bridge domains, EVPN neighbors, the ESIs, the DF PE, Aliasing label, Split Horizon label, and so on...
```
root@R1> show evpn instance extensive
Instance: EVPN-CE12
  Route Distinguisher: 10.0.0.1:1234
  VLAN ID: 1234
  Per-instance MAC route label: 16
  Duplicate MAC detection threshold: 5
  Duplicate MAC detection window: 180
  MAC database status                     Local  Remote
    MAC advertisements:                       1       3
    MAC+IP advertisements:                    0       0
    Default gateway MAC advertisements:       0       0
  Number of local interfaces: 2 (2 up)
    Interface name  ESI                            Mode             Status     AC-Role
    .local..6       00:00:00:00:00:00:00:00:00:00  single-homed     Up         Root
    ae1.1200        00:00:00:00:00:00:00:00:00:01  all-active       Up         Root
  Number of IRB interfaces: 0 (0 up)
  Number of protect interfaces: 0
  Number of bridge domains: 1
    VLAN  Domain-ID Intfs/up   IRB-intf  Mode            MAC-sync v4-SG-sync v6-SG-sync
    1234               1  1              Extended        Enabled  Disabled   Disabled
  Number of neighbors: 3
    Address               MAC    MAC+IP        AD        IM        ES Leaf-label DCI-Peer Flow-label DT2U-SID           DT2M-SID
    10.0.0.2                1         0         2         1         0                           NO
    10.0.0.3                1         0         2         1         0                           NO
    10.0.0.4                1         0         2         1         0                           NO
  Number of ethernet segments: 2
    ESI: 00:00:00:00:00:00:00:00:00:01
      Status: Resolved by IFL ae1.1200
      State-Bitfield: 0x43
      ESI Route Label: 21
      ESI Refcount: 1
      ESI Num Macs: 1, ESI Num SGDBs: 0
      Token-Route NH: 0
      Number of Local interfaces: 1
      Local interface: ae1.1200, Status: Up/Forwarding
      Number of remote PEs connected: 1
        Remote-PE        MAC-label  Aliasing-label  Mode
        10.0.0.2         22         22              all-active
      DF Election Algorithm: MOD based
      Designated forwarder: 10.0.0.1
      Backup forwarder: 10.0.0.2
      Last designated forwarder update: Apr 10 12:40:43
      Advertised MAC label: 22
      Advertised aliasing label: 22
      Advertised split horizon label: 23
    ESI: 00:00:00:00:00:00:00:00:00:02
      Status: Resolved by NH 1048649
      State-Bitfield: 0x1
      ESI Route Label: 157
      ESI Refcount: 1
      ESI Num Macs: 1, ESI Num SGDBs: 0
      Token-Route NH: 1048649
      Number of remote PEs connected: 2
        Remote-PE        MAC-label  Aliasing-label  Mode
        10.0.0.4         22         22              all-active
        10.0.0.3         22         22              all-active
  SMET Forwarding: Disabled
  RIB Table-ID: 184549386, Kernel Table-ID: 6, Kernel Table-Generation: 3
  EVPN instance flags: 0x1810008
  RTT Update Timestamp: NA
  L2ALD state change Timestamp: Mar  6 15:08:12.063 2026
  Core-Isolation change TS: Apr 10 12:40:25.560 2026, Core-Isolated: N
  Last Core-Isolation Change Reason: bgp-peer-transition

Instance: __default_evpn__
  Route Distinguisher: 10.0.0.1:0
  Number of bridge domains: 0
  Number of neighbors: 3
    Address               MAC    MAC+IP        AD        IM        ES Leaf-label DCI-Peer Flow-label DT2U-SID           DT2M-SID
    10.0.0.2                0         0         0         0         1                           NO
    10.0.0.3                0         0         0         0         1                           NO
    10.0.0.4                0         0         0         0         1                           NO
```
EVPN is the most complete and robust solution we currently have in networking, and I absolutely love it.

Now, let's establish L3 connectivity. We'll use the EVPN Virtual Gateway Address. This allows the CE to use any local PE as its default gateway, and through this, we can also route traffic to the internet! The Virtual Gateway Address feature allows us to provision an Anycast Gateway while still keeping a unique physical IP on the IRB for troubleshooting and local pinging.

The configuration is very simple: create an IRB interface with a unique IP address, and attach the Anycast Virtual Gateway address. Then, bind the routing interface to the EVPN instance and instruct BGP not to advertise this IP with the default-gateway community (to avoid host route pollution).

R1:
```
set interfaces irb unit 1234 family inet address 10.16.12.1/24 virtual-gateway-address 10.16.12.254

set routing-instances EVPN-CE12 routing-interface irb.1234
set routing-instances EVPN-CE12 protocols evpn default-gateway no-gateway-community
```
R2:
```
set interfaces irb unit 1234 family inet address 10.16.12.2/24 virtual-gateway-address 10.16.12.254

set routing-instances EVPN-CE12 routing-interface irb.1234
set routing-instances EVPN-CE12 protocols evpn default-gateway no-gateway-community
```
R3:
```
set interfaces irb unit 1234 family inet address 10.16.34.3/24 virtual-gateway-address 10.16.34.254

set routing-instances EVPN-CE34 routing-interface irb.1234
set routing-instances EVPN-CE34 protocols evpn default-gateway no-gateway-community
```
R4:
```
set interfaces irb unit 1234 family inet address 10.16.34.4/24 virtual-gateway-address 10.16.34.254

set routing-instances EVPN-CE34 routing-interface irb.1234
set routing-instances EVPN-CE34 protocols evpn default-gateway no-gateway-community
```

With the Anycast Gateways configured, let's verify our routing table:
```
root@R1> show route table EVPN-CE12.evpn.0 match-prefix 2:*

EVPN-CE12.evpn.0: 38 destinations, 38 routes (38 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2:10.0.0.1:1234::1234::00:00:5e:00:01:01/304 MAC/IP
                   *[EVPN/170] 00:23:13
                       Indirect
2:10.0.0.1:1234::1234::2c:6b:f5:48:61:f0/304 MAC/IP
                   *[EVPN/170] 00:23:13
                       Indirect
2:10.0.0.1:1234::1234::50:ea:05:00:1c:00/304 MAC/IP
                   *[EVPN/170] 03:31:50
                       Indirect
2:10.0.0.2:1234::1234::00:00:5e:00:01:01/304 MAC/IP
                   *[BGP/170] 00:23:07, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 172
2:10.0.0.2:1234::1234::2c:6b:f5:89:bd:f0/304 MAC/IP
                   *[BGP/170] 00:27:43, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 16
2:10.0.0.2:1234::1234::50:ea:05:00:1c:00/304 MAC/IP
                   *[BGP/170] 03:31:50, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 22
2:10.0.0.3:1234::1234::00:00:5e:00:01:01/304 MAC/IP
                   *[BGP/170] 00:22:53, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 427, Push 35(top)
                       to 10.200.0.3 via ge-0/0/2.0, Push 427, Push 36(top)
2:10.0.0.3:1234::1234::2c:6b:f5:34:ea:f0/304 MAC/IP
                   *[BGP/170] 00:27:22, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 16, Push 35(top)
                       to 10.200.0.3 via ge-0/0/2.0, Push 16, Push 36(top)
2:10.0.0.3:1234::1234::50:a8:c0:00:2a:00/304 MAC/IP
                   *[BGP/170] 03:31:48, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 22, Push 35(top)
                       to 10.200.0.3 via ge-0/0/2.0, Push 22, Push 36(top)
2:10.0.0.4:1234::1234::00:00:5e:00:01:01/304 MAC/IP
                   *[BGP/170] 00:22:50, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 202
2:10.0.0.4:1234::1234::2c:6b:f5:0e:34:f0/304 MAC/IP
                   *[BGP/170] 00:26:48, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 16
2:10.0.0.4:1234::1234::50:a8:c0:00:2a:00/304 MAC/IP
                   *[BGP/170] 03:31:44, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 22
2:10.0.0.1:1234::1234::00:00:5e:00:01:01::10.16.12.254/304 MAC/IP
                   *[EVPN/170] 00:23:13
                       Indirect
2:10.0.0.1:1234::1234::2c:6b:f5:48:61:f0::10.16.12.1/304 MAC/IP
                   *[EVPN/170] 00:23:13
                       Indirect
2:10.0.0.1:1234::1234::50:ea:05:00:1c:00::10.16.12.12/304 MAC/IP
                   *[EVPN/170] 00:23:09
                       Indirect
2:10.0.0.2:1234::1234::00:00:5e:00:01:01::10.16.12.254/304 MAC/IP
                   *[BGP/170] 00:23:07, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 172
2:10.0.0.2:1234::1234::2c:6b:f5:89:bd:f0::10.16.12.2/304 MAC/IP
                   *[BGP/170] 00:24:57, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 16
2:10.0.0.2:1234::1234::50:ea:05:00:1c:00::10.16.12.12/304 MAC/IP
                   *[BGP/170] 00:16:54, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 22
2:10.0.0.3:1234::1234::00:00:5e:00:01:01::10.16.34.254/304 MAC/IP
                   *[BGP/170] 00:22:53, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 427, Push 35(top)
                       to 10.200.0.3 via ge-0/0/2.0, Push 427, Push 36(top)
2:10.0.0.3:1234::1234::2c:6b:f5:34:ea:f0::10.16.34.3/304 MAC/IP
                   *[BGP/170] 00:24:43, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 16, Push 35(top)
                       to 10.200.0.3 via ge-0/0/2.0, Push 16, Push 36(top)
2:10.0.0.3:1234::1234::50:a8:c0:00:2a:00::10.16.34.34/304 MAC/IP
                   *[BGP/170] 00:16:16, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 22, Push 35(top)
                       to 10.200.0.3 via ge-0/0/2.0, Push 22, Push 36(top)
2:10.0.0.4:1234::1234::00:00:5e:00:01:01::10.16.34.254/304 MAC/IP
                   *[BGP/170] 00:22:50, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 202
2:10.0.0.4:1234::1234::2c:6b:f5:0e:34:f0::10.16.34.4/304 MAC/IP
                   *[BGP/170] 00:22:50, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 16
2:10.0.0.4:1234::1234::50:a8:c0:00:2a:00::10.16.34.34/304 MAC/IP
                   *[BGP/170] 00:16:16, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 22
```
Now we successfully have the MAC and MAC/IP routes imported from both CEs across all PEs. Everything looks fine, let's ask the customer to check the L3 connectivity between the sites:
```
[admin@CE34-1] > tool traceroute 10.16.12.12
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS      LOSS  SENT  LAST   AVG    BEST  WORST  STD-DEV
1  10.16.34.4   0%       4  4.2ms  107.3  4.2   393.4  165.4
2  10.200.0.2   0%       4  7.8ms  169.6  7.8   625.6  263.5
3  10.16.12.12  0%       4  2.3ms  10.3   2.3   21.3   7.1
```
And... it's solid! Now, the final step: providing internet access to the customer.

First, we'll create a VRF on R3 to begin the L3-EVPN interworking. We'll have two interfaces connecting to our CGNAT: one acting as the "inside" interface (LAN/VRF facing) and the other as the "outside" interface (WAN/Global routing facing).
```
set interfaces ge-0/0/7 description to-CGN-1
set interfaces ge-0/0/7 flexible-vlan-tagging
-----LAN-----
set interfaces ge-0/0/7 unit 50 description LAN
set interfaces ge-0/0/7 unit 50 vlan-id 50
set interfaces ge-0/0/7 unit 50 family inet address 172.16.3.9/30

set policy-options policy-statement Saida-CGN-VRF term 1 from route-filter 10.16.12.0/24 exact
set policy-options policy-statement Saida-CGN-VRF term 1 from route-filter 10.16.34.0/24 exact
set policy-options policy-statement Saida-CGN-VRF term 1 then next-hop self
set policy-options policy-statement Saida-CGN-VRF term 1 then accept
set policy-options policy-statement Saida-CGN-VRF then reject

set routing-instances VRF-EVPN instance-type vrf
set routing-instances VRF-EVPN protocols bgp group iBGP-CGN-AS65020 type internal
set routing-instances VRF-EVPN protocols bgp group iBGP-CGN-AS65020 description iBGP-CGN-AS65020
set routing-instances VRF-EVPN protocols bgp group iBGP-CGN-AS65020 export Saida-CGN-VRF
set routing-instances VRF-EVPN protocols bgp group iBGP-CGN-AS65020 neighbor 172.16.3.10 family inet unicast
set routing-instances VRF-EVPN description VRF-EVPN
set routing-instances VRF-EVPN interface ge-0/0/7.50
set routing-instances VRF-EVPN interface irb.1234
set routing-instances VRF-EVPN route-distinguisher 10.0.0.3:12341
set routing-instances VRF-EVPN vrf-target target:65020:12341
set routing-instances VRF-EVPN vrf-table-label
-----WAN-----
set interfaces ge-0/0/7 unit 51 description WAN
set interfaces ge-0/0/7 unit 51 vlan-id 51
set interfaces ge-0/0/7 unit 51 family inet address 172.16.3.13/30

set policy-options policy-statement Entrada-CGN term 1 from route-filter 200.0.0.0/24 exact
set policy-options policy-statement Entrada-CGN term 1 then community add Customer
set policy-options policy-statement Entrada-CGN term 1 then accept
set policy-options policy-statement Entrada-CGN then reject
set policy-options policy-statement Saida-CGN term default from protocol aggregate
set policy-options policy-statement Saida-CGN term default from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Saida-CGN term default then next-hop self
set policy-options policy-statement Saida-CGN term default then accept
set policy-options policy-statement Saida-CGN then reject

set protocols bgp group iBGP-CGN-AS65020 type internal
set protocols bgp group iBGP-CGN-AS65020 description iBGP-CGN-AS65020
set protocols bgp group iBGP-CGN-AS65020 import Entrada-CGN
set protocols bgp group iBGP-CGN-AS65020 family inet unicast
set protocols bgp group iBGP-CGN-AS65020 export Saida-CGN
set protocols bgp group iBGP-CGN-AS65020 neighbor 172.16.3.14
```
Here, R3 advertises a default route globally and receives the public prefix from the CGNAT. The CGNAT then turns around and advertises a default route back to R3, but this time into the VRF context. R3 takes that default route and stitches it into the EVPN domain as a Type 5 route. You got it? I know, it’s a beautiful dance!

Let's check the BGP advertisements to verify:
```
root@R3> show route advertising-protocol bgp 172.16.3.10

VRF-EVPN.inet.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.16.34.0/24           Self                         100        I

root@R3> show route receive-protocol bgp 172.16.3.10

VRF-EVPN.inet.0: 12 destinations, 29 routes (12 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 0.0.0.0/0               172.16.3.10                  100        I

```
Here we are cleanly advertising the local route of the IRB interface. If you are following the architecture, you know we still have one missing piece across the core.

Let's check the global RIB as well:
```
root@R3> show route advertising-protocol bgp 172.16.3.14

inet.0: 240 destinations, 248 routes (224 active, 0 holddown, 17 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 0.0.0.0/0               Self                         100        I

root@R3> show route receive-protocol bgp 172.16.3.14

inet.0: 240 destinations, 248 routes (224 active, 0 holddown, 17 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 200.0.0.0/24            172.16.3.14                  100        I

```
Here, we are successfully advertising the default route generated by the aggregate, and receiving the public prefix from the CGNAT, which we tag with the customer community before propagating it to our peerings.

Now, you might be asking yourself if you just need to configure standard L3VPN VRFs on the other PEs to resolve everything. You wouldn't be completely wrong, but we're doing things differently here. We won't use the inet-vpn family to exchange these routes across the core; we'll use pure evpn (Type 5 routes)!

First, we need to provision the VRF structure on the remaining PEs:
```
set routing-instances VRF-EVPN instance-type vrf
set routing-instances VRF-EVPN description VRF-EVPN
set routing-instances VRF-EVPN interface irb.1234
set routing-instances VRF-EVPN route-distinguisher 10.0.0.1:12341
set routing-instances VRF-EVPN vrf-target target:65020:12341
set routing-instances VRF-EVPN vrf-table-label
```

To highlight the difference between this EVPN-stitched VRF and a legacy L3VPN VRF, consider how Junos displays the routing tables. In L3VPN, if you look at a VRF without specifying inet.0 or bgp.l3vpn.0, Junos displays both: the primary table (bgp.l3vpn.0) and the secondary table where the routes are properly injected.
```
root@R1> show route table VRF-C2-SPOKE detail

VRF-C2-SPOKE.inet.0: 20 destinations, 25 routes (19 active, 0 holddown, 6 hidden)
10.2.0.3/32 (2 entries, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.0.0.4:201
....
                Communities: target:65020:201 src-as:65020:0 rt-import:10.0.0.4:10
....
                Primary Routing Table: bgp.l3vpn.0
....
```
But in L3-EVPN, we exchange the routes directly through the evpn family. Let's look at the routing table now:
```
root@R1> show route table VRF-EVPN. detail

VRF-EVPN.inet.0: 6 destinations, 13 routes (6 active, 0 holddown, 0 hidden)
0.0.0.0/0 (2 entries, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.0.0.3:12341
....
                Communities: target:65020:12341
....
                Primary Routing Table: bgp.l3vpn.0
....
         EVPN   Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0x1ccf0614
                Next-hop reference count: 3
                Kernel Table Id: 0
                Next hop type: Router, Next hop index: 0
                Next hop: 10.200.0.1 via ae0.0 weight 0x1, selected
                Label operation: Push 35
                Label TTL action: no-prop-ttl
                Load balance label: Label 35: None;
                Label element ptr: 0x8d88358
                Label parent element ptr: 0x0
                Label element references: 13
                Label element child references: 11
                Label element lsp id: 0
                Session Id: 0
                Next hop: 10.200.0.3 via ge-0/0/2.0 weight 0xf000
                Label operation: Push 36
                Label TTL action: no-prop-ttl
                Load balance label: Label 36: None;
                Label element ptr: 0x1c8e1030
                Label parent element ptr: 0x0
                Label element references: 13
                Label element child references: 11
                Label element lsp id: 0
                Session Id: 0
                Protocol next hop: 10.0.0.3
                Label operation: Push 504
                Label TTL action: no-prop-ttl
                Load balance label: Label 504: None;
                Composite next hop: 0x1c8a1d18 - INH Session ID: 0
                Composite next hop: CNH non-key opaque: 0x0, CNH key opaque: 0x84a5fd0
                Indirect next hop: 0x80e4350 - INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Int Ext Changed>
                Inactive reason: Route Metric, BGP vs. non-BGP
                Age: 5:49       Metric2: 15
                Validation State: unverified
                Localpref: 100
                Task: VRF-EVPN-EVPN-L3-context
                AS path: I  (Atomic Originator)
                Aggregator: 65020 10.0.0.3
                Cluster list:  0.0.0.2
                Originator ID: 10.0.0.3
                Thread: junos-main

VRF-EVPN.evpn.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
5:10.0.0.3:12341::0::0.0.0.0::0/248 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.0.0.3:12341
                Next hop type: Indirect, Next hop index: 0
                Address: 0x1ccee414
                Next-hop reference count: 6
                Kernel Table Id: 0
                Source: 10.0.0.0
                Protocol next hop: 10.0.0.3
                Label operation: Push 504
                Label TTL action: prop-ttl
                Load balance label: Label 504: None;
                Indirect next hop: 0x2 no-forward INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Secondary Active Int Ext>
                Local AS: 65020 Peer AS: 65020
                Age: 10:37      Metric2: 15
                Validation State: unverified
                Task: BGP_65020.10.0.0.0
                Announcement bits (1): 0-VRF-EVPN-EVPN-L3-context
                AS path: I  (Atomic Originator)
                Aggregator: 65020 10.0.0.3
                Cluster list:  0.0.0.2
                Originator ID: 10.0.0.3
                Communities: target:65020:12341
                Import Accepted
                Route Label: 504
                Overlay gateway address: 0.0.0.0
                ESI 00:00:00:00:00:00:00:00:00:00
                Localpref: 100
                Router ID: 10.0.0.0
                Primary Routing Table: bgp.evpn.0
                Thread: junos-main
```
With the Type 5 route now handling the L3 prefix, the natural next step is to completely exclude the inet-vpn family from the backbone. To test the reliability and prove EVPN can carry the routing alone, let's deactivate the inet-vpn session across the core and run a continuous ping.
```
[admin@CE12-1] > ping 10.16.34.34
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 10.16.34.34                                56  62 3ms838us
    1 10.16.34.34                                56  62 4ms39us
    2 10.16.34.34                                56  62 17ms644us
    3 10.16.34.34                                56  62 2ms454us
    4 10.16.34.34                                56  62 4ms348us
    5 10.16.34.34                                56  62 5ms234us
    6 10.16.34.34                                56  62 5ms157us
    7 10.16.34.34                                56  62 8ms772us
    8 10.16.34.34                                56  62 4ms48us
    9 10.16.34.34                                56  62 4ms749us
   10 10.16.34.34                                                  timeout
   11 10.16.34.34                                56  62 3ms100us
   12 10.16.34.34                                56  62 6ms434us
   13 10.16.34.34                                56  62 6ms315us
   14 10.16.34.34                                56  62 3ms541us
   15 10.16.34.34                                56  62 9ms149us
   16 10.16.34.34                                56  62 5ms488us
    sent=17 received=16 packet-loss=5% min-rtt=2ms454us avg-rtt=5ms894us max-rtt=17ms644us

root@R1> show route 10.16.34.34
....
VRF-EVPN.inet.0: 7 destinations, 15 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.16.34.34/32     *[BGP/170] 00:00:20, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 274
                    [BGP/170] 00:00:21, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 504, Push 35(top)
                       to 10.200.0.3 via ge-0/0/2.0, Push 504, Push 36(top)
....
root@R1> configure
root@R1# edit protocols bgp group iBGP-AS65020-West
[edit protocols bgp group iBGP-AS65020-West]
root@R1# deactivate family inet-vpn
[edit protocols bgp group iBGP-AS65020-West]
root@R1# commit and-quit
....
root@R1> show route 10.16.34.34
....
VRF-EVPN.inet.0: 5 destinations, 6 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.16.34.0/24      *[EVPN/170] 00:00:02
                    >  to 10.200.0.1 via ae0.0, Push 504, Push 35(top)
                       to 10.200.0.3 via ge-0/0/2.0, Push 504, Push 36(top)
```
Here we can see a perfect example of a live migration from L3VPN to L3-EVPN. The customer lost only a single packet during convergence, and we immediately see the EVPN Type 5 route installed as the active path in the RIB.

Now, let's verify internet access for CE12:
```
[admin@CE12-1] > tool traceroute 200.1.0.1
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS      LOSS  SENT  LAST    AVG   BEST  WORST  STD-DEV
1  10.16.12.2   0%      18  4.9ms   11.2  1.8   91     20.1
2  172.16.3.9   0%      18  8.5ms   28.9  3.1   311.9  70
3  172.16.3.10  0%      18  1.9ms   3.7   1.9   7.9    1.5
4  172.16.3.13  0%      18  5.5ms   16.7  4     93.7   21.2
5  10.200.0.23  0%      18  8.8ms   34.3  7.8   256.9  55
6  200.1.0.1    0%      18  13.1ms  17.6  8     65.9   12.6
```
Perfectly routed exclusively through EVPN!

If we want to optimize routing further, we can leak the MAC/IP routes directly into the VRF. Follow my logic here: both R1 and R2 have L2 connectivity with CE12-1. Both of them have the 10.16.12.0/24 subnet configured on their IRB interfaces, and both are exporting this /24 prefix up to R3:
```
root@R3> show route 10.16.12.0/24

inet.0: 234 destinations, 241 routes (225 active, 0 holddown, 9 hidden)

VRF-EVPN.inet.0: 7 destinations, 8 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.16.12.0/24      *[EVPN/170] 00:00:10
                    >  to 10.200.0.6 via ge-0/0/3.0, Push 240
                       to 10.200.0.11 via ge-0/0/4.0, Push 240, Push 35(top)
                    [EVPN/170] 00:05:39
                    >  to 10.200.0.6 via ge-0/0/3.0, Push 250, Push 33(top)
                       to 10.200.0.11 via ge-0/0/4.0, Push 250, Push 33(top)
```
If the physical connection between R1 and the CE fails, the IRB interface itself will not go down, meaning R1 will continue to export the /24 prefix, potentially blackholing traffic. To solve this issue and establish optimal forwarding, we can export the specific MAC/IP routes into the VRF.

We just need to add a simple knob inside the EVPN routing instances:
```
set routing-instances EVPN-CE12 protocols evpn remote-ip-host-routes
set routing-instances EVPN-CE34 protocols evpn remote-ip-host-routes
```
This knob tells the PE to leak the MAC/IP routes from the evpn RIB down into the inet RIB of the VRF. Note that this does not generate a Type 5 route; it’s simply a local process that installs the host route (/32) into the local routing table to optimize forwarding.
```
root@R3> show route 10.16.12.12
....
VRF-EVPN.inet.0: 7 destinations, 8 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.16.12.0/24      *[EVPN/170] 00:03:58
                    >  to 10.200.0.6 via ge-0/0/3.0, Push 240
                       to 10.200.0.11 via ge-0/0/4.0, Push 240, Push 35(top)
                    [EVPN/170] 00:01:03
                    >  to 10.200.0.6 via ge-0/0/3.0, Push 250, Push 33(top)
                       to 10.200.0.11 via ge-0/0/4.0, Push 250, Push 33(top)
....
set routing-instances EVPN-CE34 protocols evpn remote-ip-host-routes
....
root@R3> show route 10.16.12.12
....
VRF-EVPN.inet.0: 10 destinations, 11 routes (10 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.16.12.12/32     *[EVPN/7] 00:01:15
                    >  via irb.1234
....
root@R3> show route table EVPN-CE34.evpn.0 match-prefix *10.16.12.12* detail

EVPN-CE34.evpn.0: 36 destinations, 36 routes (36 active, 0 holddown, 0 hidden)
2:10.0.0.1:1234::1234::50:ea:05:00:1c:00::10.16.12.12/304 MAC/IP (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.0.0.1:1234
....
                Communities: target:65020:1234
                Import Accepted
                Route Label: 287
                ESI: 00:00:00:00:00:00:00:00:00:01
                Localpref: 100
                Router ID: 10.0.0.0
                Primary Routing Table: bgp.evpn.0
                Thread: junos-main

2:10.0.0.2:1234::1234::50:ea:05:00:1c:00::10.16.12.12/304 MAC/IP (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.0.0.2:1234
....
                Communities: target:65020:1234
                Import Accepted
                Route Label: 277
                ESI: 00:00:00:00:00:00:00:00:00:01
                Localpref: 100
                Router ID: 10.0.0.0
                Primary Routing Table: bgp.evpn.0
                Thread: junos-main
```
By doing this, we can actively route traffic to a specific /32 host IP through the L2-EVPN foundation!

All of our goals were accomplished today! I really enjoyed writing this article and exploring the incredible features of EVPN. I hope you enjoyed reading it too.

See you soon in the next post about Inter-AS VPNs! Bye.
