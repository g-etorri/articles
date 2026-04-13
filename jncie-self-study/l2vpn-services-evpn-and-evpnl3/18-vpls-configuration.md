# VPLS Configuration

Hello guys, I hope everyone is doing well today! I'm back with another article, and this time we're diving into VPLS!

Here is the topology we'll be using today:
<img width="1928" height="1148" alt="image" src="https://github.com/user-attachments/assets/52d5e7af-1bef-48f5-ac4f-1d18b949e8e9" />
You might notice that we have two multi-homed sites. I set it up this way to show you exactly how to handle them. We can configure multi-homed sites in VPLS signaled via both LDP and BGP.

First, let's map out the configuration with this table:
| Customer | Site | Router | Protocol | PE-CE Interface | VLAN | PE |
| ------- | ---- | ------ | ----------- | ------------------- | -------- | ------ |
| C5      | S4   | CE5-4  | BGP         | ge-0/0/8            | 500, 501 | R2     |
| C5      | S5   | CE5-5  | BGP         | ge-0/0/8, ge-0/0/7  | 500, 501 | R3, R4 |
| C5      | S6   | CE5-6  | BGP         | ge-0/0/9            | 500, 501 | R5     |
| C6      | S1   | CE6-1  | LDP         | ge-0/0/9            | 600, 601 | R1     |
| C6      | S2   | CE6-2  | LDP         | ge-0/0/5, ge-0/0/8  | 600, 601 | R8, R7 |
| C6      | S3   | CE6-3  | LDP         | ge-0/0/6            | 600, 601 | R6     |

Let's start by configuring the VPLS for Customer 5 using the Kompella draft. This time we don't need to touch the backbone configuration; the VPLS will use the l2vpn BGP family to exchange information.

Another detail to note: on each connection, we have two VLANs to maintain two separate broadcast domains. We'll configure the VPLS in VLAN-AWARE mode to keep these broadcast domains isolated within the VPLS. Let's get to it.

Starting on R2, here is the configuration:
```
set interfaces ge-0/0/8 description to-CE5-4
set interfaces ge-0/0/8 flexible-vlan-tagging
set interfaces ge-0/0/8 encapsulation flexible-ethernet-services
set interfaces ge-0/0/8 unit 500 encapsulation vlan-vpls
set interfaces ge-0/0/8 unit 500 vlan-id 500
set interfaces ge-0/0/8 unit 501 encapsulation vlan-vpls
set interfaces ge-0/0/8 unit 501 vlan-id 501

set routing-instances VPLS-CE5 instance-type vpls
set routing-instances VPLS-CE5 protocols vpls site s4 interface ge-0/0/8.500
set routing-instances VPLS-CE5 protocols vpls site s4 interface ge-0/0/8.501
set routing-instances VPLS-CE5 protocols vpls site s4 site-identifier 4
set routing-instances VPLS-CE5 protocols vpls label-block-size 8
set routing-instances VPLS-CE5 protocols vpls no-tunnel-services
set routing-instances VPLS-CE5 description VPLS-CE5
set routing-instances VPLS-CE5 vlan-id all
set routing-instances VPLS-CE5 interface ge-0/0/8.500
set routing-instances VPLS-CE5 interface ge-0/0/8.501
set routing-instances VPLS-CE5 route-distinguisher 10.0.0.2:500
set routing-instances VPLS-CE5 vrf-target target:65020:500

set protocols bgp group iBGP-AS65020-East family l2vpn signaling
```
Note: The ```vlan-id all``` command configures the VPLS for VLAN-aware mode. You'll notice some similar elements to what we used in Kompella VPWS. The ```no-tunnel-services``` knob tells the router to dynamically create an LSI/VT interface, avoiding the need for dedicated physical loopback (lt) interfaces.

We can apply this same logic across the other PEs in the network. However, to avoid Layer 2 loops, the configuration for multi-homed sites requires special attention. Let's check the VPLS connections first, and then we'll focus on configuring the multi-homed sites.

In our BGP table, we can see the route received from Site 6:
```
root@R2> show route table VPLS-CE5.l2vpn.0 detail

VPLS-CE5.l2vpn.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
 10.0.0.2:500:4:1/96 (1 entry, 1 announced)
        *L2VPN  Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0x809a094
                Next-hop reference count: 11
                Kernel Table Id: 0
                Protocol next hop: 10.0.0.2
                Indirect next hop: 0x0 - INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Active Int Ext>
                Age: 4w4d 19:43:59      Metric2: 1
                Validation State: unverified
                Task: VPLS-CE5-l2vpn
                Announcement bits (1): 1-BGP_RT_Background
                AS path: I
                Communities: Layer2-info: encaps: VPLS, control flags:[0x0] , mtu: 0, site preference: 100
                Label-base: 24, range: 8, status-vector: 0xEB, offset: 1
                Thread: junos-main

 10.0.0.5:500:6:1/96 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.0.0.5:500
                Next hop type: Indirect, Next hop index: 0
                Address: 0x1cca8694
                Next-hop reference count: 7
                Kernel Table Id: 0
                Source: 10.0.0.0
                Protocol next hop: 10.0.0.5
                Indirect next hop: 0x2 no-forward INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Secondary Active Int Ext>
                Local AS: 65020 Peer AS: 65020
                Age: 1d 18:25:14        Metric2: 1
                Validation State: unverified
                Task: BGP_65020.10.0.0.0
                Announcement bits (1): 0-VPLS-CE5-l2vpn
                AS path: I  (Originator)
                Cluster list:  0.0.0.2
                Originator ID: 10.0.0.5
                Communities: target:65020:500 Layer2-info: encaps: VPLS, control flags:[0x0] , mtu: 0, site preference: 100
                Import Accepted
                Label-base: 19, range: 8, offset: 1
                Localpref: 100
                Router ID: 10.0.0.0
                Primary Routing Table: bgp.l2vpn.0
                Thread: junos-main
```
Here we can see the label-base, the block range, and the offset. The router uses this data to calculate which label is needed to forward traffic to this specific site. With this information, Junos automatically provisions the VPLS connection:
```
root@R2> show vpls connections
Layer-2 VPN connections:

Legend for connection status (St)
EI -- encapsulation invalid      NC -- interface encapsulation not CCC/TCC/VPLS
EM -- encapsulation mismatch     WE -- interface and instance encaps not same
VC-Dn -- Virtual circuit down    NP -- interface hardware not present
CM -- control-word mismatch      -> -- only outbound connection is up
CN -- circuit not provisioned    <- -- only inbound connection is up
OR -- out of range               Up -- operational
OL -- no outgoing label          Dn -- down
LD -- local site signaled down   CF -- call admission control failure
RD -- remote site signaled down  SC -- local and remote site ID collision
LN -- local site not designated  LM -- local site ID not minimum designated
RN -- remote site not designated RM -- remote site ID not minimum designated
XX -- unknown connection status  IL -- no incoming label
MM -- MTU mismatch               MI -- Mesh-Group ID not available
BK -- Backup connection          ST -- Standby connection
PF -- Profile parse failure      PB -- Profile busy
RS -- remote site standby        SN -- Static Neighbor
LB -- Local site not best-site   RB -- Remote site not best-site
VM -- VLAN ID mismatch           HS -- Hot-standby Connection

Legend for interface status
Up -- operational
Dn -- down

Instance: VPLS-CE5
Edge protection: Not-Primary
  Local site: s4 (4)
    connection-site           Type  St     Time last up          # Up trans
    6                         rmt   Up     Apr  6 16:28:37 2026           1
      Remote PE: 10.0.0.5, Negotiated control-word: No
      Incoming label: 29, Outgoing label: 22
      Local interface: vt-0/0/0.1048576, Status: Up, Encapsulation: VPLS
        Description: Intf - vpls VPLS-CE5 local site 4 remote site 6
      Flow Label Transmit: No, Flow Label Receive: No
```
The output shows the flags Junos might set if the circuit encounters an issue—very similar to the l2circuit flags we saw in the previous article. Now, for the main difference between VPLS and VPWS: in VPWS, there is no MAC learning and you logically only have one peer. In VPLS, MAC learning is enabled! This allows the router to learn the source MAC addresses and dynamically forward traffic across a multi-point topology. Let's check the VPLS MAC table:
```
root@R2> show vpls mac-table

MAC flags       (S -static MAC, D -dynamic MAC, L -locally learned, C -Control MAC
    O -OVSDB MAC, SE -Statistics enabled, NM -Non configured MAC, R -Remote PE MAC, P -Pinned MAC)

Routing instance : VPLS-CE5
 Bridging domain : __VPLS-CE5__, VLAN : 500
   MAC                 MAC      GBP     Logical          NH     MAC         active
   address             flags    Tag     interface        Index  property    source
   50:9d:8c:00:25:00   D                ge-0/0/8.500
   50:d6:06:00:32:00   D                vt-0/0/0.1048576

MAC flags       (S -static MAC, D -dynamic MAC, L -locally learned, C -Control MAC
    O -OVSDB MAC, SE -Statistics enabled, NM -Non configured MAC, R -Remote PE MAC, P -Pinned MAC)

Routing instance : VPLS-CE5
 Bridging domain : __VPLS-CE5__, VLAN : 501
   MAC                 MAC      GBP     Logical          NH     MAC         active
   address             flags    Tag     interface        Index  property    source
   50:9d:8c:00:25:00   D                ge-0/0/8.501
   50:d6:06:00:32:00   D                vt-0/0/0.1048576
```
Notice the ```vt``` interface in the output. This is essentially a virtual interface that represents the pushing of the VPLS label. Another crucial detail: there are two separate MAC tables! This confirms that our VLAN-aware mode is working perfectly, isolating the broadcast domains.

Before we ask the customer to run tests, let's wrap up the service provisioning by configuring the multi-homed site.

If you remember your JNCIP-SP studies, multi-homing in Kompella VPLS isn't active-active; it operates in an active-standby mode. Only one PE will actively forward traffic to the customer site, and the active forwarder is elected based on the BGP local-preference value attached to the routes. Let's see it in action.

On R3, the core logic of the VPLS remains the same, but we are introducing the ```multi-homing``` and ```site-preference``` knobs under the VPLS protocol hierarchy:
```
set interfaces ge-0/0/8 description to-CE5-5
set interfaces ge-0/0/8 flexible-vlan-tagging
set interfaces ge-0/0/8 encapsulation flexible-ethernet-services
set interfaces ge-0/0/8 unit 500 encapsulation vlan-vpls
set interfaces ge-0/0/8 unit 500 vlan-id 500
set interfaces ge-0/0/8 unit 501 encapsulation vlan-vpls
set interfaces ge-0/0/8 unit 501 vlan-id 501

set routing-instances VPLS-CE5 instance-type vpls
set routing-instances VPLS-CE5 protocols vpls site s5 multi-homing
set routing-instances VPLS-CE5 protocols vpls site s5 site-preference primary
set routing-instances VPLS-CE5 protocols vpls site s5 interface ge-0/0/8.500
set routing-instances VPLS-CE5 protocols vpls site s5 interface ge-0/0/8.501
set routing-instances VPLS-CE5 protocols vpls site s5 site-identifier 5
set routing-instances VPLS-CE5 protocols vpls label-block-size 8
set routing-instances VPLS-CE5 protocols vpls no-tunnel-services
set routing-instances VPLS-CE5 description VPLS-CE5
set routing-instances VPLS-CE5 vlan-id all
set routing-instances VPLS-CE5 interface ge-0/0/8.500
set routing-instances VPLS-CE5 interface ge-0/0/8.501
set routing-instances VPLS-CE5 route-distinguisher 10.0.0.3:500
set routing-instances VPLS-CE5 vrf-target target:65020:500
set protocols bgp group iBGP-AS65020-East family l2vpn signaling
```
On R4, we'll do the exact same thing, but we'll change the ```site-preference``` to ```backup```:
```
set interfaces ge-0/0/6 description to-CE5-5
set interfaces ge-0/0/6 flexible-vlan-tagging
set interfaces ge-0/0/6 encapsulation flexible-ethernet-services
set interfaces ge-0/0/6 unit 500 encapsulation vlan-vpls
set interfaces ge-0/0/6 unit 500 vlan-id 500
set interfaces ge-0/0/6 unit 501 encapsulation vlan-vpls
set interfaces ge-0/0/6 unit 501 vlan-id 501

set routing-instances VPLS-CE5 instance-type vpls
set routing-instances VPLS-CE5 protocols vpls site s5 multi-homing
set routing-instances VPLS-CE5 protocols vpls site s5 site-preference backup
set routing-instances VPLS-CE5 protocols vpls site s5 interface ge-0/0/6.500
set routing-instances VPLS-CE5 protocols vpls site s5 interface ge-0/0/6.501
set routing-instances VPLS-CE5 protocols vpls site s5 site-identifier 5
set routing-instances VPLS-CE5 protocols vpls label-block-size 8
set routing-instances VPLS-CE5 protocols vpls no-tunnel-services
set routing-instances VPLS-CE5 description VPLS-CE5
set routing-instances VPLS-CE5 vlan-id all
set routing-instances VPLS-CE5 interface ge-0/0/6.500
set routing-instances VPLS-CE5 interface ge-0/0/6.501
set routing-instances VPLS-CE5 route-distinguisher 10.0.0.4:500
set routing-instances VPLS-CE5 vrf-target target:65020:500
```
Now, let's inspect the routes:
```
root@R2> show route table VPLS-CE5.l2vpn.0 detail match-prefix *:500:5:1*

VPLS-CE5.l2vpn.0: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
 10.0.0.3:500:5:1/96 (1 entry, 1 announced)
        *BGP    Preference: 170/-65536
                Route Distinguisher: 10.0.0.3:500
                Next hop type: Indirect, Next hop index: 0
                Address: 0x1cca8e94
                Next-hop reference count: 11
                Kernel Table Id: 0
                Source: 10.0.0.0
                Protocol next hop: 10.0.0.3
                Indirect next hop: 0x2 no-forward INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Secondary Active Int Ext>
                Local AS: 65020 Peer AS: 65020
                Age: 7:14       Metric2: 1
                Validation State: unverified
                Task: BGP_65020.10.0.0.0
                Announcement bits (1): 0-VPLS-CE5-l2vpn
                AS path: I  (Originator)
                Cluster list:  0.0.0.2
                Originator ID: 10.0.0.3
                Communities: target:65020:500 Layer2-info: encaps: VPLS, control flags:[0x0] , mtu: 0, site preference: 65535
                Import Accepted
                Label-base: 291, range: 8, offset: 1
                Localpref: 65535
                Router ID: 10.0.0.0
                Primary Routing Table: bgp.l2vpn.0
                Thread: junos-main

 10.0.0.4:500:5:1/96 (1 entry, 1 announced)
        *BGP    Preference: 170/-2
                Route Distinguisher: 10.0.0.4:500
                Next hop type: Indirect, Next hop index: 0
                Address: 0x80b2b14
                Next-hop reference count: 8
                Kernel Table Id: 0
                Source: 10.0.0.0
                Protocol next hop: 10.0.0.4
                Indirect next hop: 0x2 no-forward INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Secondary Active Int Ext>
                Local AS: 65020 Peer AS: 65020
                Age: 7:14       Metric2: 1
                Validation State: unverified
                Task: BGP_65020.10.0.0.0
                Announcement bits (1): 0-VPLS-CE5-l2vpn
                AS path: I  (Originator)
                Cluster list:  0.0.0.2
                Originator ID: 10.0.0.4
                Communities: target:65020:500 Layer2-info: encaps: VPLS, control flags:[0x0] , mtu: 0, site preference: 1
                Import Accepted
                Label-base: 148, range: 8, offset: 1
                Localpref: 1
                Router ID: 10.0.0.0
                Primary Routing Table: bgp.l2vpn.0
                Thread: junos-main
```
Notice that the ```site-preference``` translates directly into the BGP local-preference. This allows remote routers to determine which PE is the designated forwarder for that site. Now, let's verify the connections on R3 and R4!

On R3, we can see that the connections to the other two sites are in the Up state, but strangely, Site 5 also appears in the list as a remote connection! This happens because R3 is receiving the VPLS route generated by R4 (which also represents Site 5). However, note the RN flag (Remote Site Not Designated). This indicates that R4 is not the active traffic forwarder for this site.
```
root@R3> show vpls connections
Layer-2 VPN connections:

Legend for connection status (St)
EI -- encapsulation invalid      NC -- interface encapsulation not CCC/TCC/VPLS
EM -- encapsulation mismatch     WE -- interface and instance encaps not same
VC-Dn -- Virtual circuit down    NP -- interface hardware not present
CM -- control-word mismatch      -> -- only outbound connection is up
CN -- circuit not provisioned    <- -- only inbound connection is up
OR -- out of range               Up -- operational
OL -- no outgoing label          Dn -- down
LD -- local site signaled down   CF -- call admission control failure
RD -- remote site signaled down  SC -- local and remote site ID collision
LN -- local site not designated  LM -- local site ID not minimum designated
RN -- remote site not designated RM -- remote site ID not minimum designated
XX -- unknown connection status  IL -- no incoming label
MM -- MTU mismatch               MI -- Mesh-Group ID not available
BK -- Backup connection          ST -- Standby connection
PF -- Profile parse failure      PB -- Profile busy
RS -- remote site standby        SN -- Static Neighbor
LB -- Local site not best-site   RB -- Remote site not best-site
VM -- VLAN ID mismatch           HS -- Hot-standby Connection

Legend for interface status
Up -- operational
Dn -- down

Instance: VPLS-CE5
Edge protection: Not-Primary
  Local site: s5 (5)
    connection-site           Type  St     Time last up          # Up trans
    4                         rmt   Up     Apr  8 11:07:28 2026           1
      Remote PE: 10.0.0.2, Negotiated control-word: No
      Incoming label: 294, Outgoing label: 28
      Local interface: vt-0/0/0.1048832, Status: Up, Encapsulation: VPLS
        Description: Intf - vpls VPLS-CE5 local site 5 remote site 4
      Flow Label Transmit: No, Flow Label Receive: No
    5                         rmt   RN
    6                         rmt   Up     Apr  8 11:07:28 2026           1
      Remote PE: 10.0.0.5, Negotiated control-word: No
      Incoming label: 296, Outgoing label: 23
      Local interface: vt-0/0/0.1048833, Status: Up, Encapsulation: VPLS
        Description: Intf - vpls VPLS-CE5 local site 5 remote site 6
      Flow Label Transmit: No, Flow Label Receive: No
```
Now, let's check R4: 
```
root@R4> show vpls connections
Layer-2 VPN connections:

Legend for connection status (St)
EI -- encapsulation invalid      NC -- interface encapsulation not CCC/TCC/VPLS
EM -- encapsulation mismatch     WE -- interface and instance encaps not same
VC-Dn -- Virtual circuit down    NP -- interface hardware not present
CM -- control-word mismatch      -> -- only outbound connection is up
CN -- circuit not provisioned    <- -- only inbound connection is up
OR -- out of range               Up -- operational
OL -- no outgoing label          Dn -- down
LD -- local site signaled down   CF -- call admission control failure
RD -- remote site signaled down  SC -- local and remote site ID collision
LN -- local site not designated  LM -- local site ID not minimum designated
RN -- remote site not designated RM -- remote site ID not minimum designated
XX -- unknown connection status  IL -- no incoming label
MM -- MTU mismatch               MI -- Mesh-Group ID not available
BK -- Backup connection          ST -- Standby connection
PF -- Profile parse failure      PB -- Profile busy
RS -- remote site standby        SN -- Static Neighbor
LB -- Local site not best-site   RB -- Remote site not best-site
VM -- VLAN ID mismatch           HS -- Hot-standby Connection

Legend for interface status
Up -- operational
Dn -- down

Instance: VPLS-CE5
Edge protection: Not-Primary
  Local site: s5 (5)
    connection-site           Type  St     Time last up          # Up trans
    4                         rmt   LN
    5                         rmt   LN
    6                         rmt   LN
```
All connections on R4 show the LN flag (Local Site Not Designated). This confirms that R4 is acting as the standby and is not forwarding traffic for Site 5. This election mechanism strictly prevents any Layer 2 loops that could be caused by the multi-homed connection.

Let's check the final MAC table and ask the customer to run some pings:
```
root@R3> show vpls mac-table

MAC flags       (S -static MAC, D -dynamic MAC, L -locally learned, C -Control MAC
    O -OVSDB MAC, SE -Statistics enabled, NM -Non configured MAC, R -Remote PE MAC, P -Pinned MAC)

Routing instance : VPLS-CE5
 Bridging domain : __VPLS-CE5__, VLAN : 500
    MAC                 MAC      GBP     Logical          NH     MAC         active
   address             flags    Tag     interface        Index  property    source
   50:66:cc:00:2b:00   D                ge-0/0/8.500
   50:9d:8c:00:25:00   D                vt-0/0/0.1048832
   50:d6:06:00:32:00   D                vt-0/0/0.1048833
   aa:bb:cc:00:04:00   D                ge-0/0/8.500

MAC flags       (S -static MAC, D -dynamic MAC, L -locally learned, C -Control MAC
    O -OVSDB MAC, SE -Statistics enabled, NM -Non configured MAC, R -Remote PE MAC, P -Pinned MAC)

Routing instance : VPLS-CE5
 Bridging domain : __VPLS-CE5__, VLAN : 501
   MAC                 MAC      GBP     Logical          NH     MAC         active
   address             flags    Tag     interface        Index  property    source
   50:66:cc:00:2b:00   D                ge-0/0/8.501
   50:9d:8c:00:25:00   D                vt-0/0/0.1048832
   50:d6:06:00:32:00   D                vt-0/0/0.1048833
   aa:bb:cc:00:04:00   D                ge-0/0/8.501
```
As expected, Site 5 has full connectivity with the other sites in the network:
```
[admin@CE5-5] > ping 172.50.0.4
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.50.0.4                                 56  64 5ms896us
    1 172.50.0.4                                 56  64 18ms302us
    2 172.50.0.4                                 56  64 14ms519us
    sent=3 received=3 packet-loss=0% min-rtt=5ms896us avg-rtt=12ms905us max-rtt=18ms302us

[admin@CE5-5] > ping 172.50.0.6
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.50.0.6                                 56  64 13ms300us
    1 172.50.0.6                                 56  64 8ms974us
    2 172.50.0.6                                 56  64 9ms564us
    sent=3 received=3 packet-loss=0% min-rtt=8ms974us avg-rtt=10ms612us max-rtt=13ms300us
```
Mission accomplished! Customer 5 is successfully provisioned.

Now, let's move on to the Martini VPLS! Just like L2CKTs, the configuration here is very simple and straightforward.

I'll apply the configuration on R1, but the same logic applies to the other PEs. Again, leave the multi-homed site out for now to avoid loops.
```
set interfaces ge-0/0/9 description to-CE6-1
set interfaces ge-0/0/9 flexible-vlan-tagging
set interfaces ge-0/0/9 encapsulation flexible-ethernet-services
set interfaces ge-0/0/9 unit 600 encapsulation vlan-vpls
set interfaces ge-0/0/9 unit 600 vlan-id 600
set interfaces ge-0/0/9 unit 601 encapsulation vlan-vpls
set interfaces ge-0/0/9 unit 601 vlan-id 601

set routing-instances VPLS-CE6 instance-type vpls
set routing-instances VPLS-CE6 protocols vpls neighbor 10.0.0.6
set routing-instances VPLS-CE6 protocols vpls encapsulation-type ethernet-vlan
set routing-instances VPLS-CE6 protocols vpls no-tunnel-services
set routing-instances VPLS-CE6 protocols vpls vpls-id 6
set routing-instances VPLS-CE6 protocols vpls mtu 1500
set routing-instances VPLS-CE6 protocols vpls flow-label-transmit
set routing-instances VPLS-CE6 protocols vpls flow-label-receive
set routing-instances VPLS-CE6 description VPLS-CE6
set routing-instances VPLS-CE6 vlan-id all
set routing-instances VPLS-CE6 interface ge-0/0/9.600
set routing-instances VPLS-CE6 interface ge-0/0/9.601
```
Here we need to match the exact same parameters we discussed for L2CKTs: encapsulation, MTU, and vpls-id must be identical on all peers, otherwise the pseudowire won't come up. I added the flow-label configuration to improve traffic load-balancing across the core, it's just a good design practice. And just like we did with Kompella, we'll use VLAN-aware mode to keep the broadcast domains separated.

Alright, with R1 and R6 configured, let's check the status:
```
root@R1> show vpls connections
Layer-2 VPN connections:

Legend for connection status (St)
EI -- encapsulation invalid      NC -- interface encapsulation not CCC/TCC/VPLS
EM -- encapsulation mismatch     WE -- interface and instance encaps not same
VC-Dn -- Virtual circuit down    NP -- interface hardware not present
CM -- control-word mismatch      -> -- only outbound connection is up
CN -- circuit not provisioned    <- -- only inbound connection is up
OR -- out of range               Up -- operational
OL -- no outgoing label          Dn -- down
LD -- local site signaled down   CF -- call admission control failure
RD -- remote site signaled down  SC -- local and remote site ID collision
LN -- local site not designated  LM -- local site ID not minimum designated
 RN -- remote site not designated RM -- remote site ID not minimum designated
XX -- unknown connection status  IL -- no incoming label
MM -- MTU mismatch               MI -- Mesh-Group ID not available
BK -- Backup connection          ST -- Standby connection
PF -- Profile parse failure      PB -- Profile busy
RS -- remote site standby        SN -- Static Neighbor
LB -- Local site not best-site   RB -- Remote site not best-site
VM -- VLAN ID mismatch           HS -- Hot-standby Connection

Legend for interface status
Up -- operational
Dn -- down

Instance: VPLS-CE6
  VPLS-id: 6
    Neighbor                  Type  St     Time last up          # Up trans
    10.0.0.6(vpls-id 6)       rmt   Up     Mar  6 15:12:06 2026           1
      Remote PE: 10.0.0.6, Negotiated control-word: No
      Incoming label: 26, Outgoing label: 17
      Negotiated PW status TLV: No
      Local interface: lsi.1048576, Status: Up, Encapsulation: VLAN
        Description: Intf - vpls VPLS-CE6 neighbor 10.0.0.6 vpls-id 6
      Flow Label Transmit: Yes, Flow Label Receive: Yes
    10.0.0.8(vpls-id 6)       rmt   OL
    10.0.0.7(vpls-id 6)       rmt   OL
```
The pseudowire to R6 is established successfully. Let's take a look under the hood at the LDP FEC 128 message:
```
root@R1> show ldp database session 10.0.0.6 l2circuit detail
Input label database, 10.0.0.1:0--10.0.0.6:0
Labels received: 11
  Label     Prefix
     17      L2CKT NoCtrlWord VLAN VC 6
            MTU: 1500 Flow Label T Bit: 1 Flow Label R Bit: 1
            State: Active
            Age: 4w4d 20:45:46
            VCCV Control Channel types:
                MPLS router alert label
                MPLS PW label with TTL=1
            VCCV Control Verification types:
                LSP ping
                BFD with IP/UDP-encapsulation for Fault Detection

Output label database, 10.0.0.1:0--10.0.0.6:0
Labels advertised: 10
  Label     Prefix
     26      L2CKT NoCtrlWord VLAN VC 6
            MTU: 1500 Flow Label T Bit: 1 Flow Label R Bit: 1
            State: Active
            Age: 4w4d 20:45:47
            VCCV Control Channel types:
                MPLS router alert label
                MPLS PW label with TTL=1
            VCCV Control Verification types:
                LSP ping
                BFD with IP/UDP-encapsulation for Fault Detection
```
In the LDP message details, we can clearly see all the negotiated parameters, including the MTU, Flow Label bits, virtual-circuit-id, and encapsulation type. This level of visibility is fantastic for troubleshooting!

Now, let's tackle the multi-homed site for Customer 6.

To ensure we only have one active traffic forwarder in the VPLS, we want the active pseudowires to point to a single router. I've chosen R8 as the primary. So, on R1 and R6, let's add this configuration:
```
set routing-instances VPLS-CE6 protocols vpls neighbor 10.0.0.8 revert-time 30
set routing-instances VPLS-CE6 protocols vpls neighbor 10.0.0.8 backup-neighbor 10.0.0.7 standby
```
This tells the router to establish pseudowires to both R8 and R7, but the pseudowire to R7 will only be used if the connection to R8 fails. The revert-time determines how long the router waits before moving traffic back to R8 after it recovers. Here, it's set to 30 seconds.

On R8 and R7, we'll configure the local VPLS instances. However, to prevent a core loop, we must not configure a pseudowire between R8 and R7 themselves.
```
set routing-instances VPLS-CE6 instance-type vpls
set routing-instances VPLS-CE6 protocols vpls interface-mac-limit 20
set routing-instances VPLS-CE6 protocols vpls neighbor 10.0.0.1
set routing-instances VPLS-CE6 protocols vpls neighbor 10.0.0.6
set routing-instances VPLS-CE6 protocols vpls encapsulation-type ethernet-vlan
set routing-instances VPLS-CE6 protocols vpls no-tunnel-services
set routing-instances VPLS-CE6 protocols vpls vpls-id 6
set routing-instances VPLS-CE6 protocols vpls mtu 1500
set routing-instances VPLS-CE6 protocols vpls flow-label-transmit
set routing-instances VPLS-CE6 protocols vpls flow-label-receive
set routing-instances VPLS-CE6 description VPLS-CE6
set routing-instances VPLS-CE6 vlan-id all
set routing-instances VPLS-CE6 interface ge-0/0/8.600
set routing-instances VPLS-CE6 interface ge-0/0/8.601
```
With that applied, let's check the connection status on R1:
```
root@R1> show vpls connections
Layer-2 VPN connections:

Legend for connection status (St)
EI -- encapsulation invalid      NC -- interface encapsulation not CCC/TCC/VPLS
EM -- encapsulation mismatch     WE -- interface and instance encaps not same
VC-Dn -- Virtual circuit down    NP -- interface hardware not present
CM -- control-word mismatch      -> -- only outbound connection is up
CN -- circuit not provisioned    <- -- only inbound connection is up
OR -- out of range               Up -- operational
OL -- no outgoing label          Dn -- down
LD -- local site signaled down   CF -- call admission control failure
RD -- remote site signaled down  SC -- local and remote site ID collision
LN -- local site not designated  LM -- local site ID not minimum designated
RN -- remote site not designated RM -- remote site ID not minimum designated
XX -- unknown connection status  IL -- no incoming label
MM -- MTU mismatch               MI -- Mesh-Group ID not available
BK -- Backup connection          ST -- Standby connection
PF -- Profile parse failure      PB -- Profile busy
RS -- remote site standby        SN -- Static Neighbor
LB -- Local site not best-site   RB -- Remote site not best-site
VM -- VLAN ID mismatch           HS -- Hot-standby Connection

Legend for interface status
Up -- operational
Dn -- down

Instance: VPLS-CE6
  VPLS-id: 6
    Neighbor                  Type  St     Time last up          # Up trans
    10.0.0.6(vpls-id 6)       rmt   Up     Mar  6 15:12:06 2026           1
      Remote PE: 10.0.0.6, Negotiated control-word: No
      Incoming label: 26, Outgoing label: 17
      Negotiated PW status TLV: No
      Local interface: lsi.1048576, Status: Up, Encapsulation: VLAN
        Description: Intf - vpls VPLS-CE6 neighbor 10.0.0.6 vpls-id 6
      Flow Label Transmit: Yes, Flow Label Receive: Yes
    10.0.0.8(vpls-id 6)       rmt   Up     Apr  8 12:06:58 2026           1
      Remote PE: 10.0.0.8, Negotiated control-word: No
      Incoming label: 24, Outgoing label: 134
      Negotiated PW status TLV: No
      Local interface: lsi.1048580, Status: Up, Encapsulation: VLAN
        Description: Intf - vpls VPLS-CE6 neighbor 10.0.0.8 vpls-id 6
      Flow Label Transmit: Yes, Flow Label Receive: Yes
    10.0.0.7(vpls-id 6)       rmt   ST
```
As you can see, the pseudowire to R8 is Up, while the pseudowire to R7 shows the ST flag, meaning it is a Standby Connection.

If you check R7, you'll see the pseudowires to R1 and R6 are established. This is exactly what the ```standby``` knob does on the remote ends: the signaling happens and the pseudowire is technically up, but the data plane is blocked and not in use.
```
root@R7> show vpls connections
Layer-2 VPN connections:

Legend for connection status (St)
EI -- encapsulation invalid      NC -- interface encapsulation not CCC/TCC/VPLS
EM -- encapsulation mismatch     WE -- interface and instance encaps not same
VC-Dn -- Virtual circuit down    NP -- interface hardware not present
CM -- control-word mismatch      -> -- only outbound connection is up
CN -- circuit not provisioned    <- -- only inbound connection is up
OR -- out of range               Up -- operational
OL -- no outgoing label          Dn -- down
LD -- local site signaled down   CF -- call admission control failure
RD -- remote site signaled down  SC -- local and remote site ID collision
LN -- local site not designated  LM -- local site ID not minimum designated
RN -- remote site not designated RM -- remote site ID not minimum designated
XX -- unknown connection status  IL -- no incoming label
MM -- MTU mismatch               MI -- Mesh-Group ID not available
BK -- Backup connection          ST -- Standby connection
PF -- Profile parse failure      PB -- Profile busy
RS -- remote site standby        SN -- Static Neighbor
LB -- Local site not best-site   RB -- Remote site not best-site
VM -- VLAN ID mismatch           HS -- Hot-standby Connection

Legend for interface status
Up -- operational
Dn -- down

Instance: VPLS-CE6
  VPLS-id: 6
    Neighbor                  Type  St     Time last up          # Up trans
    10.0.0.1(vpls-id 6)       rmt   Up     Apr  8 12:06:47 2026           1
      Remote PE: 10.0.0.1, Negotiated control-word: No
      Incoming label: 41, Outgoing label: 25
      Negotiated PW status TLV: No
      Local interface: lsi.1048832, Status: Up, Encapsulation: VLAN
        Description: Intf - vpls VPLS-CE6 neighbor 10.0.0.1 vpls-id 6
      Flow Label Transmit: Yes, Flow Label Receive: Yes
    10.0.0.6(vpls-id 6)       rmt   Up     Apr  8 12:06:47 2026           1
      Remote PE: 10.0.0.6, Negotiated control-word: No
      Incoming label: 42, Outgoing label: 19
      Negotiated PW status TLV: No
      Local interface: lsi.1048833, Status: Up, Encapsulation: VLAN
        Description: Intf - vpls VPLS-CE6 neighbor 10.0.0.6 vpls-id 6
      Flow Label Transmit: Yes, Flow Label Receive: Yes
```
Because of this standby state, R7 theoretically shouldn't learn any MAC addresses from the remote PEs:
```
root@R7> show vpls mac-table

MAC flags       (S -static MAC, D -dynamic MAC, L -locally learned, C -Control MAC
    O -OVSDB MAC, SE -Statistics enabled, NM -Non configured MAC, R -Remote PE MAC, P -Pinned MAC)

Routing instance : VPLS-CE6
 Bridging domain : __VPLS-CE6__, VLAN : 600
   MAC                 MAC      GBP     Logical          NH     MAC         active
   address             flags    Tag     interface        Index  property    source
   50:05:8f:00:33:00   D                ge-0/0/8.600
   50:32:6d:00:38:00   D                ge-0/0/8.600
   50:fc:a2:00:20:00   D                ge-0/0/8.600

MAC flags       (S -static MAC, D -dynamic MAC, L -locally learned, C -Control MAC
    O -OVSDB MAC, SE -Statistics enabled, NM -Non configured MAC, R -Remote PE MAC, P -Pinned MAC)

Routing instance : VPLS-CE6
 Bridging domain : __VPLS-CE6__, VLAN : 601
   MAC                 MAC      GBP     Logical          NH     MAC         active
   address             flags    Tag     interface        Index  property    source
   50:05:8f:00:33:00   D                ge-0/0/8.601
   50:32:6d:00:38:00   D                ge-0/0/8.601
   50:fc:a2:00:20:00   D                ge-0/0/8.601
```
Unlike Kompella VPLS, Martini VPLS doesn't have native BGP signaling between the multi-homed PEs to coordinate the site election. Because of this, R7 is still learning MAC addresses from its locally connected interface, even though it isn't forwarding them to the core.

Meanwhile, on R8 (the active forwarder), the MAC table is fully populated:
```
root@R8> show vpls mac-table

MAC flags       (S -static MAC, D -dynamic MAC, L -locally learned, C -Control MAC
    O -OVSDB MAC, SE -Statistics enabled, NM -Non configured MAC, R -Remote PE MAC, P -Pinned MAC)

Routing instance : VPLS-CE6
 Bridging domain : __VPLS-CE6__, VLAN : 600
   MAC                 MAC      GBP     Logical          NH     MAC         active
   address             flags    Tag     interface        Index  property    source
   50:05:8f:00:33:00   D                lsi.1048833
   50:32:6d:00:38:01   D                ge-0/0/5.600
   50:fc:a2:00:20:00   D                lsi.1048832

MAC flags       (S -static MAC, D -dynamic MAC, L -locally learned, C -Control MAC
    O -OVSDB MAC, SE -Statistics enabled, NM -Non configured MAC, R -Remote PE MAC, P -Pinned MAC)

Routing instance : VPLS-CE6
 Bridging domain : __VPLS-CE6__, VLAN : 601
   MAC                 MAC      GBP     Logical          NH     MAC         active
   address             flags    Tag     interface        Index  property    source
   50:05:8f:00:33:00   D                lsi.1048833
   50:32:6d:00:38:01   D                ge-0/0/5.601
   50:fc:a2:00:20:00   D                lsi.1048832
```
Alright, let's apply some constraints. I don't want this customer filling up our tables with more than 20 MAC addresses. I'll configure a limit and tell the router to drop packets if the customer exceeds it:

Let's apply this across all PEs:
```
set routing-instances VPLS-CE6 protocols vpls mac-table-size 20
set routing-instances VPLS-CE6 protocols vpls mac-table-size packet-action drop
set routing-instances VPLS-CE6 protocols vpls interface-mac-limit 20
```
If there's ever a bridging loop or an attack on the customer's LAN, this constraint prevents it from overloading our PE memory.

Let's ask the customer to run some pings to confirm the service is fully operational:
```
[admin@CE6-1] > ping 172.60.0.2
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.60.0.2                                 56  64 49ms856us
    1 172.60.0.2                                 56  64 21ms605us
    2 172.60.0.2                                 56  64 12ms477us
    sent=3 received=3 packet-loss=0% min-rtt=12ms477us avg-rtt=27ms979us max-rtt=49ms856us

[admin@CE6-1] > ping 172.60.0.3
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.60.0.3                                 56  64 28ms6us
    1 172.60.0.3                                 56  64 6ms309us
    2 172.60.0.3                                 56  64 15ms477us
    sent=3 received=3 packet-loss=0% min-rtt=6ms309us avg-rtt=16ms597us max-rtt=28ms6us
```
Everything looks great!

But wait, Customer 5 is calling me...

They want to connect Site 1 (which is currently on a VPWS) directly into their VPLS! No problem, let's make it happen.

On R7, we'll add another VLAN unit to the PE-CE interface and map it to a new L2CKT pseudowire:
```
set interfaces ge-0/0/7 unit 500 description S1-VPLS
set interfaces ge-0/0/7 unit 500 encapsulation vlan-ccc
set interfaces ge-0/0/7 unit 500 vlan-id 500
set routing-instances L2VPN-CE5-S1 protocols l2vpn site s1 interface ge-0/0/7.500 remote-site-id 10
set routing-instances L2VPN-CE5-S1 interface ge-0/0/7.500
```
This will transport all traffic from VLAN 500 over to "Site 10". Site 10 doesn't actually exist in the physical world; it's a fictitious local site we are going to create on R3 just to stitch the VPWS and VPLS together.

On R3, we need to create two units on a logical-tunnel (``lt``) interface. One unit will be placed into the VPWS routing instance, and the other into the VPLS instance. Both units will use VLAN 500, but they require different encapsulation types to translate the frames between VPWS and VPLS properly.
```
set interfaces lt-0/0/0 unit 0 encapsulation vlan-ccc
set interfaces lt-0/0/0 unit 0 vlan-id 500
set interfaces lt-0/0/0 unit 0 peer-unit 1
set interfaces lt-0/0/0 unit 1 encapsulation vlan-vpls
set interfaces lt-0/0/0 unit 1 vlan-id 500
set interfaces lt-0/0/0 unit 1 peer-unit 0

set routing-instances L2VPN-CE5-2 protocols l2vpn site s10 interface lt-0/0/0.1 remote-site-id 1
set routing-instances L2VPN-CE5-2 protocols l2vpn site s10 site-identifier 10
set routing-instances L2VPN-CE5-2 protocols l2vpn site s10 mtu 1500
set routing-instances L2VPN-CE5-2 interface lt-0/0/0.1

set routing-instances VPLS-CE5 protocols vpls site s10 interface lt-0/0/0.1
set routing-instances VPLS-CE5 protocols vpls site s10 site-identifier 10
set routing-instances VPLS-CE5 interface lt-0/0/0.1
```
Inside the VPWS configuration, we define our fictitious Site 10 and point it towards Site 1. Then, in the VPLS configuration, we simply add a new site and bind the other end of the lt interface to it.

Now, traffic arriving from Site 1 on VLAN 500 travels across the core to R3. R3 receives it on the VPWS, passes it through the logical tunnel (lt-0/0/0.0 to lt-0/0/0.1), and injects it straight into the VPLS! The reverse path works exactly the same way. See? It's elegant and easy once you get the hang of it.

While I'm at it, I'll add a MAC limit to the Customer 5 VPLS as well. I'll cap the MAC learning at 16 addresses:
```
set routing-instances VPLS-CE5 protocols vpls mac-table-size 16
set routing-instances VPLS-CE5 protocols vpls interface-mac-limit 16
```
But this time, I won't drop the packets when the limit is hit.

Let's check with the customer to ensure Site 1 can reach the rest of the VPLS sites:
```
[admin@CE5-1] > ping 172.50.0.4
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.50.0.4                                 56  64 61ms722us
    1 172.50.0.4                                 56  64 51ms805us
    2 172.50.0.4                                 56  64 20ms892us
    sent=3 received=3 packet-loss=0% min-rtt=20ms892us avg-rtt=44ms806us
   max-rtt=61ms722us

[admin@CE5-1] > ping 172.50.0.5
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.50.0.5                                 56  64 66ms469us
    1 172.50.0.5                                 56  64 26ms139us
    2 172.50.0.5                                 56  64 10ms183us
    sent=3 received=3 packet-loss=0% min-rtt=10ms183us avg-rtt=34ms263us
   max-rtt=66ms469us

[admin@CE5-1] > ping 172.50.0.6
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.50.0.6                                 56  64 56ms565us
    1 172.50.0.6                                 56  64 20ms47us
    2 172.50.0.6                                 56  64 46ms428us
    sent=3 received=3 packet-loss=0% min-rtt=20ms47us avg-rtt=41ms13us
   max-rtt=56ms565us

[admin@CE5-1] > ip arp pr
Flags: D, P - PUBLISHED; C - COMPLETE
Columns: ADDRESS, MAC-ADDRESS, INTERFACE
#    ADDRESS     MAC-ADDRESS        INTERFACE
0 DC 172.5.12.2  50:E3:AC:00:24:00  vlan512
1 DC 172.5.13.3  50:5D:ED:00:31:00  vlan513
2 DC 172.50.0.6  50:D6:06:00:32:00  vlan500
3 DC 172.50.0.5  50:66:CC:00:2B:00  vlan500
4 DC 172.50.0.4  50:9D:8C:00:25:00  vlan500
```
Everything is working flawlessly!

All of our primary goals are accomplished, but I promised a bonus! Do you remember what I mentioned in the previous article about the LDP FEC 128 message? Because Martini VPLS and L2CKTs use the exact same signaling message, we can actually establish a pseudowire where one side terminates in a VPLS and the other side terminates in a VPWS! Let me show you an example. We can use this to build an H-VPLS (Hierarchical VPLS), effectively creating a hub-and-spoke topology.

On R1 (the Hub), we'll configure a VPLS instance. On R5 and R6 (the Spokes), we'll configure standard L2CKTs.
R1:
```
set interfaces ge-0/0/9 unit 610 encapsulation vlan-vpls
set interfaces ge-0/0/9 unit 610 vlan-id 610

set routing-instances H-VPLS-Example instance-type vpls
set routing-instances H-VPLS-Example protocols vpls neighbor 10.0.0.6
set routing-instances H-VPLS-Example protocols vpls neighbor 10.0.0.5
set routing-instances H-VPLS-Example protocols vpls encapsulation-type ethernet-vlan
set routing-instances H-VPLS-Example protocols vpls no-tunnel-services
set routing-instances H-VPLS-Example protocols vpls vpls-id 610
set routing-instances H-VPLS-Example protocols vpls mtu 1500
set routing-instances H-VPLS-Example protocols vpls flow-label-transmit
set routing-instances H-VPLS-Example protocols vpls flow-label-receive
set routing-instances H-VPLS-Example description H-VPLS-Example
set routing-instances H-VPLS-Example vlan-id 610 
set routing-instances H-VPLS-Example interface ge-0/0/9.610
```
R5 and R6:
```
set interfaces ge-0/0/9 unit 610 encapsulation vlan-ccc
set interfaces ge-0/0/9 unit 610 vlan-id 610
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/9.610 virtual-circuit-id 610
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/9.610 description L2CKT-610
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/9.610 flow-label-transmit
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/9.610 flow-label-receive
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/9.610 mtu 1500
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/9.610 encapsulation-type ethernet-vlan
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/9.610 no-vlan-id-validate
```

Now, checking R1, we see the pseudowires established natively into the VPLS:
```
root@R1> show vpls connections instance H-VPLS-Example
Layer-2 VPN connections:

Legend for connection status (St)
EI -- encapsulation invalid      NC -- interface encapsulation not CCC/TCC/VPLS
EM -- encapsulation mismatch     WE -- interface and instance encaps not same
VC-Dn -- Virtual circuit down    NP -- interface hardware not present
CM -- control-word mismatch      -> -- only outbound connection is up
CN -- circuit not provisioned    <- -- only inbound connection is up
OR -- out of range               Up -- operational
OL -- no outgoing label          Dn -- down
LD -- local site signaled down   CF -- call admission control failure
RD -- remote site signaled down  SC -- local and remote site ID collision
LN -- local site not designated  LM -- local site ID not minimum designated
RN -- remote site not designated RM -- remote site ID not minimum designated
XX -- unknown connection status  IL -- no incoming label
MM -- MTU mismatch               MI -- Mesh-Group ID not available
BK -- Backup connection          ST -- Standby connection
PF -- Profile parse failure      PB -- Profile busy
RS -- remote site standby        SN -- Static Neighbor
LB -- Local site not best-site   RB -- Remote site not best-site
VM -- VLAN ID mismatch           HS -- Hot-standby Connection

Legend for interface status
Up -- operational
Dn -- down

Instance: H-VPLS-Example
  VPLS-id: 610
    Neighbor                  Type  St     Time last up          # Up trans
    10.0.0.5(vpls-id 610)     rmt   VC-Dn  -----                          0
      Remote PE: 10.0.0.5, Negotiated control-word: No
      Incoming label: 145, Outgoing label: 149
      Negotiated PW status TLV: No
      Local interface: lsi.1048593, Status: Up, Encapsulation: VLAN
        Description: Intf - vpls H-VPLS-Example neighbor 10.0.0.5 vpls-id 610
      Flow Label Transmit: Yes, Flow Label Receive: Yes
    10.0.0.6(vpls-id 610)     rmt   Up     Apr  8 12:36:22 2026           1
      Remote PE: 10.0.0.6, Negotiated control-word: No
      Incoming label: 146, Outgoing label: 117
      Negotiated PW status TLV: No
      Local interface: lsi.1048592, Status: Up, Encapsulation: VLAN
        Description: Intf - vpls H-VPLS-Example neighbor 10.0.0.6 vpls-id 610
      Flow Label Transmit: Yes, Flow Label Receive: Yes
```
Because this is naturally a hub-and-spoke topology, R5 and R6 cannot communicate directly with each other by default (due to VPLS split-horizon rules). If we wanted them to communicate through the Hub, we would need to place both pseudowires into a mesh-group and enable local-switching on R1 to alter the default forwarding behavior.

That's all for today, folks! In the next article, we are finally diving into EVPN, an incredibly exciting and modern topic! See you next time.
