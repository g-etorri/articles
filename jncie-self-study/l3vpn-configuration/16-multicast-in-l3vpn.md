# Multicast Configuration in L3VPN

Hello y'all, today is the day that I'm resisting for. Today we need to configure multicast both on our network, as a TV service, and on Customer VPN! 

You already know the topology:
<img width="1146" height="822" alt="image" src="https://github.com/user-attachments/assets/d7e82fbe-fd3a-4abb-872e-a6ff15d2a601" />

First, I'll configure the MVPN for the Customer 2! Why the MVPN first? This way I can show you that MVPN do not depends on PIM in our core interfaces. MVPN only depends on BGP to signaling the multicast streams. 

Ok, let's start enabling the mvpn signaling both on RR and on PEs. 
```
set protocols bgp group iBGP-AS65020-West family inet-mvpn signaling nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-West family inet-mvpn signaling no-install
set protocols bgp group iBGP-AS65020-East family inet-mvpn signaling nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-East family inet-mvpn signaling no-install
```
We can apply the same logic on the other routers. 

Now, let's follow to the MVPN configuration. With a L3VPN working, we can use the same routing-instance to roll out the MVPN. 

To start, let me give us a important information. The customer will use the multicast range 239.0.0.0/24. The groups 239.0.0.1 e 239.0.0.2 must be using selective PMSIs, and the others groups inclusive PMSIs, with this, we can follow the road map. 

Do you remember the topology of Customer 2? We have a hub-and-spoke topology, and R1 and R2 are the HUB PEs. In this case, the HUB PEs will act as C-RP (Customer`s RP) and will be the sender sites. The SPOKE PEs will be the receiver sites. To forward this traffic to the SPOKE PEs, we'll use MLSPs, or point-to-multipoint LSPs! During the configuration, I'll define some constraints to learn something more. 

Starting, let's configure the MVPN on the HUB routing-instance: 
* On R1 and R2 we'll use a new loopback address (10.2.0.254), acting as anycast C-RP. 
* We need to configure the PIM on PE-CE interface to have communication with the customer, and define the RP address of the group range 239.0.0.0/24 used by the customer. 
* The MVPN define the MVPN, sure. Here we are defining that our PE will be the sender site and by default, , and we defined different RTs for the MVPN.
* We created a P2MP template to use as LSPs of our MVPN.
* Finally, we need to add the chassis configuration to de-encapsulate the PIM messages.
```
set interfaces lo0 unit 2 family inet address 10.2.1.253/32 primary
set interfaces lo0 unit 2 family inet address 10.2.0.254/32

set routing-instances VRF-C2-SPOKE protocols pim rp local address 10.2.0.254
set routing-instances VRF-C2-SPOKE protocols pim rp local group-ranges 239.0.0.0/24
set routing-instances VRF-C2-SPOKE protocols pim interface all

set routing-instances VRF-C2-SPOKE protocols mvpn sender-site
set routing-instances VRF-C2-SPOKE protocols mvpn route-target import-target target target:65020:204
set routing-instances VRF-C2-SPOKE protocols mvpn route-target export-target target target:65020:204

set protocols mpls label-switched-path lsp-mcast-p2mp-template template
set protocols mpls label-switched-path lsp-mcast-p2mp-template bandwidth 10
set protocols mpls label-switched-path lsp-mcast-p2mp-template priority 5 5
set protocols mpls label-switched-path lsp-mcast-p2mp-template hop-limit 5
set protocols mpls label-switched-path lsp-mcast-p2mp-template link-protection
set protocols mpls label-switched-path lsp-mcast-p2mp-template p2mp

set routing-instances VRF-C2-HUB provider-tunnel rsvp-te label-switched-path-template lsp-mcast-p2mp-template

set chassis fpc 0 pic 0 tunnel-services
```
Note: Here we are applying the configuration on the VRF-SPOKE, but it doesn't matter, the MVPN will be established with the RTs defined in the protocols mvpn configuration. Another topic, we can use the PIM-SM in our network to signal de PMSIs, but I don't want this here, let's use the BGP and MPLS configured to make this case cleaner. 

I applied some TE constraints into LSP template to make the things funny, 10M of BW because we'll have a selective PMSI configured later to forward the surplus traffic. 

This same configuration was applied in R1 and R2. 

In the SPOKE PEs, the configuration will be more simple:
```
set routing-instances VRF-C2-SPOKE protocols mvpn receiver-site
set routing-instances VRF-C2-SPOKE protocols mvpn mvpn-mode spt-only
set routing-instances VRF-C2-SPOKE protocols mvpn route-target import-target target target:65020:204
set routing-instances VRF-C2-SPOKE protocols mvpn route-target export-target target target:65020:204

set routing-instances VRF-C2-SPOKE protocols pim interface all
```

With this, we can see the Intra-AS I-PMSI routes, or MVPN type 1 routes. 
```
root@R1> show route table VRF-C2-HUB.mvpn   

VRF-C2-HUB.mvpn.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.0.0.1:200:10.0.0.1/240               
                   *[MVPN/70] 1d 19:14:44, metric2 1
                       Indirect
1:10.0.0.2:200:10.0.0.2/240               
                   *[BGP/170] 1d 19:10:34, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 0
1:10.0.0.4:201:10.0.0.4/240               
                   *[BGP/170] 1d 19:06:37, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 0
1:10.0.0.5:201:10.0.0.5/240               
                   *[BGP/170] 1d 19:06:17, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 10106, Push 10107, Push 10108(top)
                       to 10.200.0.5 via ge-0/0/3.0, Push 10106, Push 10107, Push 10108(top)
1:10.0.0.7:201:10.0.0.7/240               
                   *[BGP/170] 1d 19:06:04, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 45
....
VRF-C2-HUB.mvpn.0: 7 destinations, 8 routes (7 active, 0 holddown, 0 hidden)
1:10.0.0.1:200:10.0.0.1/240 (1 entry, 1 announced)
        *MVPN   Preference: 70
                PMSI: Flags 0x0: Label 0: RSVP-TE: Session_13[10.0.0.1:0:60993:10.0.0.1]
                Next hop type: Indirect, Next hop index: 0
                Address: 0x8099f14
                Next-hop reference count: 9
                Kernel Table Id: 0
                Protocol next hop: 10.0.0.1
                Indirect next hop: 0x0 - INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Active Int Ext>
                Age: 22:41:48   Metric2: 1 
                Validation State: unverified 
                Task: mvpn global task
                Announcement bits (3): 0-PIM.VRF-C2-HUB 1-mvpn global task 2-rt-export 
                AS path: I 
                Thread: junos-main 
```
Note: The route format of MVPN route type 1 is ROUTE-TYPE:ROUTE-DISTINGUISHER:ORIGIN-ROUTER. In this route, will contain the PMSI type used, in our case is RSVP-TE. 
This route is used for autodiscovery of the MVPN. Do you remember the MVPN routes? I don't. 

I'll give you a table with the routes:

| Route Type | Name | Who Sends? | Description |
| - | - | - | - |
| Type 1 | Intra-AS I-PMSI | All PEs | Used for MVPN autodiscovery |
| Type 2 | Inter-AS I-PMSI | ASBRs | Used in Inter-AS VPNs scenario, Option B or C |
| Type 3 | S-PMSI AD | Sender PE | Used to create a selective tunnel for a particular group |
| Type 4 | Leaf AD | Receiver PE | Used to respond the route type 3, to build a selective tree |
| Type 5 | Source Active | Sender PE | Used to inform the PEs that the source started the stream |
| Type 6 | Shared Tree Join | Receiver PE | Used to join a shared tree, in other words, to receive the stream from RP |
| Type 7 | Source Tree Join | Receiver PE | Used to join a source based tree, the PE wants to receive the stream from the source |

This table will facilitate our understanding about MVPN. 

Now, checking our MVPN instance, we can see the members:
```
root@R1> show mvpn instance 

MVPN instance:
Legend for neighbor state (St)
A-    Preferred upstream neighbor for inter-AS

Legend for provider tunnel
S-    Selective provider tunnel
F-    Flood NH forwarding NH
M-    Multicast Composite NH
C-    Cloned NH

Legend for c-multicast routes properties (St)
DS -- derived from (*, c-g)          RM -- remote VPN route
I -- Inactive
Family : INET
    
Instance : VRF-C2-HUB
  MVPN Mode : SPT-ONLY
  Sender-Based RPF: Disabled. Reason: Not enabled by configuration.
  Hot Root Standby: Disabled. Reason: Not enabled by configuration.
  Provider tunnel: I-P-tnl:RSVP-TE P2MP:10.0.0.1, 60991,10.0.0.1
  Neighbor                      Inclusive Provider Tunnel                           Label-In    St             Segment
  10.0.0.2                              
  10.0.0.4              
  10.0.0.5              
  10.0.0.7                        
```
And, we can see the PIM neighbor of the instance:
```
root@R1> show pim neighbors instance VRF-C2-HUB 
B = Bidirectional Capable, G = Generation Identifier
H = Hello Option Holdtime, L = Hello Option LAN Prune Delay,
P = Hello Option DR Priority, T = Tracking Bit,
A = Hello Option Join Attribute

Instance: PIM.VRF-C2-HUB
Interface           IP V Mode        Option       Uptime Neighbor addr
ge-0/0/8.200         4 2             HPLGT    1d 19:02:08 10.2.1.2          
```

Ok, now we are ready to forward the multicast traffic. Let's create a stream on our network and some members of the group. I'll use a tool of Mikrotik, where I can create a traffic flow of a determinated packet, in other words, I can create a multicast stream here. 
```
[admin@CE2-1] /tool/traffic-generator> packet-template/print 
 0 name="multicast" header-stack=mac,ip,udp interface=vlan200 assumed-mac-src=50:C2:E7:00:1F:00 
   assumed-mac-dst=FF:FF:FF:FF:FF:FF assumed-mac-protocol=ip assumed-ip-dscp=0 assumed-ip-id=0 assumed-ip-frag-off=0 
   assumed-ip-ttl=64 ip-dst=239.0.0.1 assumed-ip-src=10.2.1.2 assumed-ip-protocol=udp assumed-udp-src-port=100 
   assumed-udp-dst-port=200 assumed-udp-checksum=0 data=uninitialized data-byte=0 random-byte-offsets-and-masks="" 
   random-ranges="" special-footer=yes compute-checksum-from-offset=no-checksum

[admin@CE2-1] /tool/traffic-generator> quick tx-template=multicast mbps=1 
Columns: SEQ, ID, TX-PACKET, TX-RATE, RX-PACKET, RX-RATE, RX-OOO, RX-BAD-CSUM, LOST-PACKET, LOST-RATE, LOST-RATIO
SEQ  ID  TX-PACKET  TX-RATE     RX-PACKET  RX-RATE  RX-OOO  RX-BAD-CSUM  LOST-PACKET  LOST-RATE   LOST-RATIO
1     0         83  1005.2kbps          0  0bps          0            0           83  1005.2kbps   100.000% 
2     0         83  1005.2kbps          0  0bps          0            0           83  1005.2kbps   100.000% 
TOT   0        166  1005.2kbps          0  0bps          0            0          166  1005.2kbps   100.000% 
```
When the stream is started, the R1 will create a route type 5, to advertise that a stream has started here, let's see the route below: 
```
root@R1> show route table VRF-C2-HUB.mvpn.0 match-prefix 5:* extensive  

VRF-C2-HUB.mvpn.0: 7 destinations, 8 routes (7 active, 0 holddown, 0 hidden)
5:10.0.0.1:200:32:10.2.1.2:32:239.0.0.1/240 (1 entry, 1 announced)
        *PIM    Preference: 105
                Next hop type: Multicast (IPv4) Composite, Next hop index: 0
                Address: 0x6fd9350
                Next-hop reference count: 7
                Kernel Table Id: 0
                State: <Active Int>
                Age: 2:18:28 
                Validation State: unverified 
                Task: PIM.VRF-C2-HUB
                Announcement bits (3): 0-PIM.VRF-C2-HUB 1-mvpn global task 2-rt-export 
                AS path: I 
                Communities: mvpn-sa-rp:10.2.0.254:0
                Thread: junos-main 
```
You can see the route inform us, the route type, router-id, the source of the stream and the receiver group. In other words, R1 is receiving a multicast stream from 10.2.1.2 to 239.0.0.1! 

Now, let's include a IGMP client here. Our receiver-sites will receive a IGMP join and translate this into route type 7, and advertise this to our MVPN. With this route received by R1, the traffic will be forwarded to the PE receiver. Let's check this. 

In the CE2-3, I created a manual GMP join, in the group 239.0.0.1 to stream with source 10.2.1.2. 
```
/routing gmp
add disabled=no groups=239.0.0.1 interfaces=vlan201 sources=10.2.1.2
```
R4 will receive this join:
```
root@R4> show igmp group 239.0.0.1 detail 
Interface: ge-0/0/9.201, Groups: 3
    Group: 239.0.0.1
        Group mode: Include
        Source: 10.2.1.2
        Source timeout: 173
        Last reported by: 10.2.4.2
        Group timeout:       0 Type: Dynamic
        Output interface: ge-0/0/9.201
```
As the interface speaks PIM, this will be a PIM join also:
```
root@R4> show pim join instance VRF-C2-SPOKE extensive 
Instance: PIM.VRF-C2-SPOKE Family: INET
R = Rendezvous Point Tree, S = Sparse, W = Wildcard

Group: 239.0.0.1
    Source: 10.2.1.2
    Flags: sparse,spt
    Upstream protocol: BGP
    Upstream interface: Through BGP           
    Upstream neighbor: Through MVPN
    Upstream state: Join to Source
    Keepalive timeout: 51
    Uptime: 01:47:39 
    Downstream neighbors:
        Interface: ge-0/0/9.201           
            10.2.4.1 State: Join Flags: S   Timeout: Infinity
            Uptime: 01:39:49 Time since last Join: 01:39:49
            10.2.4.2 State: Join Flags: S Timeout: 171
            Uptime: 01:47:39 Time since last Join: 00:00:39
    Number of downstream interfaces: 1
    Number of downstream neighbors: 2
```
This PIM join will be translated into route type 7 of MVPN, this route basically informs to the MVPN that R4 are interested into receive this stream: 
```
root@R4> show route table VRF-C2-SPOKE.mvpn.0 match-prefix 7:* extensive 

VRF-C2-SPOKE.mvpn.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
7:10.0.0.1:200:65020:32:10.2.1.2:32:239.0.0.1/240 (1 entry, 1 announced)
        *PIM    Preference: 105
                Next hop type: Multicast (IPv4) Composite, Next hop index: 1048575
                Address: 0x91b0324
                Next-hop reference count: 4
                Kernel Table Id: 0
                State: <Active Int Ext>
                Age: 7  Metric: 0 
                Validation State: unverified 
                Task: PIM.VRF-C2-SPOKE
                Announcement bits (3): 0-PIM.VRF-C2-SPOKE 1-mvpn global task 2-rt-export 
                AS path: I 
                Communities: target:10.0.0.1:9
                Thread: junos-main 
```
You can note the RT target:10.0.0.1:9, this is a dynamic RT created by Junos when we add the protocols mvpn into the routing-instance. All the routes will have this RT, including the inet-vpn prefixes. 

When R1 receives this route, a new PIM join is created, and the downstream interface will be the MVPN neighbors interested in receive this traffic. 
```
root@R1> show pim join instance VRF-C2-HUB extensive                      
Instance: PIM.VRF-C2-HUB Family: INET
R = Rendezvous Point Tree, S = Sparse, W = Wildcard

Group: 239.0.0.1
    Source: 10.2.1.2
    Flags: sparse,spt
    Upstream interface: ge-0/0/8.200          
    Upstream neighbor: Direct
    Upstream state: Local Source, Local RP
    Keepalive timeout: 227
    Uptime: 00:02:13 
    Downstream neighbors:
        Interface: Pseudo-MVPN            
            Uptime: 00:02:12 Time since last Join: 00:02:12
    Number of downstream interfaces: 1
    Number of downstream neighbors: 1
```
Let's check if the traffic are arriving into receiver-sites:
```
[admin@CE2-3] > interface/monitor-traffic aggregate 
        rx-packets-per-second:         82
           rx-bits-per-second:  993.1kbps
     fp-rx-packets-per-second:          0
        fp-rx-bits-per-second:       0bps
          rx-drops-per-second:          0
         rx-errors-per-second:          0
        tx-packets-per-second:          1
           tx-bits-per-second:    2.7kbps
     fp-tx-packets-per-second:          0
        fp-tx-bits-per-second:       0bps
          tx-drops-per-second:          0
    tx-queue-drops-per-second:          0
         tx-errors-per-second:          0
...
[admin@CE2-4] > interface/monitor-traffic aggregate 
        rx-packets-per-second:         80
           rx-bits-per-second:  968.9kbps
     fp-rx-packets-per-second:          0
        fp-rx-bits-per-second:       0bps
          rx-drops-per-second:          0
         rx-errors-per-second:          0
        tx-packets-per-second:          0
           tx-bits-per-second:       0bps
     fp-tx-packets-per-second:          0
        fp-tx-bits-per-second:       0bps
          tx-drops-per-second:          0
    tx-queue-drops-per-second:          0
         tx-errors-per-second:          0
...
[admin@CE2-5] > interface/monitor-traffic aggregate 
        rx-packets-per-second:         82
           rx-bits-per-second:  981.5kbps
     fp-rx-packets-per-second:          0
        fp-rx-bits-per-second:       0bps
          rx-drops-per-second:          0
         rx-errors-per-second:          0
        tx-packets-per-second:          0
           tx-bits-per-second:       0bps
     fp-tx-packets-per-second:          0
        fp-tx-bits-per-second:       0bps
          tx-drops-per-second:          0
    tx-queue-drops-per-second:          0
         tx-errors-per-second:          0
```
And... it's ok! Our MVPN is working perfectly. 

But, let's add some constraints here. I'll start another stream to the group 239.0.0.2, but in this case we'll create a selective PMSI, in other words, this PMSI will be used for a specific group and source. 

I'll limit the number of PMSIs that can be signalled in 4, and I'll create another template and apply some TE constraints in this LSP. 
```
set routing-instances VRF-C2-HUB provider-tunnel selective tunnel-limit 4
set routing-instances VRF-C2-HUB provider-tunnel selective group 239.0.0.1/32 source 10.2.0.0/16 rsvp-te label-switched-path-template lsp-mcast-selec-p2mp-template
set routing-instances VRF-C2-HUB provider-tunnel selective group 239.0.0.1/32 source 10.2.0.0/16 threshold-rate 500

et protocols mpls label-switched-path lsp-mcast-selec-p2mp-template template
set protocols mpls label-switched-path lsp-mcast-selec-p2mp-template bandwidth 60m
set protocols mpls label-switched-path lsp-mcast-selec-p2mp-template priority 5 5
set protocols mpls label-switched-path lsp-mcast-selec-p2mp-template hop-limit 5
set protocols mpls label-switched-path lsp-mcast-selec-p2mp-template link-protection
set protocols mpls label-switched-path lsp-mcast-selec-p2mp-template p2mp
```
Our LSP will signal 60Mbps of BW, will have a priority value of 5, link-protection desired and can have a limit of 5 hops, like the template of inclusive PMSI. 
The selective PMSI will be signalled when the multicast stream exceed the threshold defined, that is 500kbps. When the traffic exceed this, the sender PE will advertise a route type 3, and the receiver PE will advertise a route type 4 as a responde, with this, the selective tree will be build and the selective PMSI will be signalled. 

Now, let's check the results.

I'm still sending the stream with 1Mbps 
```
[admin@CE2-1] /tool/traffic-generator> quick tx-template=multicast mbps=1
Columns: SEQ, ID, TX-PACKET, TX-RATE, RX-PACKET, RX-RATE, RX-OOO, RX-BAD-CSUM, LOST-PACKET, LOST-RATE, LOST-RATIO
SEQ  ID  TX-PACKET  TX-RATE     RX-PACKET  RX-RATE  RX-OOO  RX-BAD-CSUM  LOST-PACKET  LOST-RATE   LOST-RATIO
149   0         82  993.1kbps           0  0bps          0            0           82  993.1kbps    100.000% 
150   0         83  1005.2kbps          0  0bps          0            0           83  1005.2kbps   100.000% 
151   0         83  1005.2kbps          0  0bps          0            0           83  1005.2kbps   100.000% 
```
This traffic is sufficient to trigger the creation of the selective tree, so, by the logic, I'll have the route type 3 created and advertised to the MVPN. 
```
root@R1> show route table bgp.mvpn.0 match-prefix 3:* extensive    

bgp.mvpn.0: 11 destinations, 12 routes (11 active, 0 holddown, 0 hidden)
3:10.0.0.1:200:32:10.2.1.2:32:239.0.0.1:10.0.0.1/240 (1 entry, 1 announced)
TSI:
Page 0 idx 0, (group iBGP-AS65020-West type Internal) Type 1 val 0x1ca4f4e0 (adv_entry)
   Advertised metrics:
     Flags: Nexthop Change
     Nexthop: Self
     Localpref: 100
     AS path: [65020] I
     Communities: target:65020:204
     PMSI: Flags 0x1: Label 0: RSVP-TE: Session_13[10.0.0.1:0:60994:10.0.0.1]
    Advertise: 00000001
Path 3:10.0.0.1:200:32:10.2.1.2:32:239.0.0.1:10.0.0.1
Vector len 4.  Val: 0
        *MVPN   Preference: 70
                PMSI: Flags 0x1: Label 0: RSVP-TE: Session_13[10.0.0.1:0:60994:10.0.0.1]
                Next hop type: Indirect, Next hop index: 0
                Address: 0x8099f14
                Next-hop reference count: 11
                Kernel Table Id: 0
                Protocol next hop: 10.0.0.1
                Indirect next hop: 0x0 - INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Secondary Active Int Ext>
                Age: 3:22       Metric2: 1 
                Validation State: unverified 
                Task: mvpn global task
                Announcement bits (1): 1-BGP_RT_Background 
                AS path: I 
                Communities: target:65020:204
                Primary Routing Table: VRF-C2-HUB.mvpn.0
                Thread: junos-main
...
root@R1> ...protocol bgp 10.0.0.0 table bgp.mvpn.0 detail                   

bgp.mvpn.0: 11 destinations, 12 routes (11 active, 0 holddown, 0 hidden)
* 1:10.0.0.1:200:10.0.0.1/240 (1 entry, 1 announced)
 BGP group iBGP-AS65020-West type Internal
     Route Distinguisher: 10.0.0.1:200
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [65020] I 
     Communities: target:65020:204

* 3:10.0.0.1:200:32:10.2.1.2:32:239.0.0.1:10.0.0.1/240 (1 entry, 1 announced)
 BGP group iBGP-AS65020-West type Internal
     Route Distinguisher: 10.0.0.1:200
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [65020] I 
     Communities: target:65020:204
```
Note: Here you can see in the flags and in the tunnel identifier the difference. In the PMSI advertised into route type 1, we have the flags 0x0, here we have another flags 0x1, and the identifier was 10.0.0.1:0:60993:10.0.0.1, now is 10.0.0.1:0:60994:10.0.0.1. 

With this route advertised, we should receive the routes type 4 of the receiver PEs, let's check:
```
root@R1> show route table bgp.mvpn.0 match-prefix 4:* extensive             

bgp.mvpn.0: 11 destinations, 12 routes (11 active, 0 holddown, 0 hidden)
4:3:10.0.0.1:200:32:10.2.1.2:32:239.0.0.1:10.0.0.1:10.0.0.4/240 (1 entry, 0 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0x80aba14
                Next-hop reference count: 4
                Kernel Table Id: 0
                Source: 10.0.0.0
                Protocol next hop: 10.0.0.4
                Indirect next hop: 0x2 no-forward INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Active Int Ext>
                Local AS: 65020 Peer AS: 65020
                Age: 6:54       Metric2: 10 
                Validation State: unverified 
                Task: BGP_65020.10.0.0.0
                AS path: I  (Originator)
                Cluster list:  0.0.0.2
                Originator ID: 10.0.0.4
                Communities: target:10.0.0.1:0
                Import Accepted
                Localpref: 100          
                Router ID: 10.0.0.0
                Secondary Tables: VRF-C2-HUB.mvpn.0
                Thread: junos-main 
                Indirect next hops: 1
                        Protocol next hop: 10.0.0.4 Metric: 10 ResolvState: Resolved
                        Indirect next hop: 0x2 no-forward INH Session ID: 0
                        Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                        Indirect path forwarding next hops: 1
                                Next hop type: Router
                                Next hop: 10.200.0.3 via ge-0/0/2.0 weight 0x1
                                Session Id: 0
                                10.0.0.4/32 Originating RIB: inet.3
                                  Metric: 10 Node path count: 1
                                  Forwarding nexthops: 1
                                        Next hop type: Router
                                        Next hop: 10.200.0.3 via ge-0/0/2.0 weight 0x1
                                        Session Id: 0

4:3:10.0.0.1:200:32:10.2.1.2:32:239.0.0.1:10.0.0.1:10.0.0.5/240 (1 entry, 0 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0x809f094
                Next-hop reference count: 4
                Kernel Table Id: 0
                Source: 10.0.0.0
                Protocol next hop: 10.0.0.5
                Indirect next hop: 0x2 no-forward INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Active Int Ext>
                Local AS: 65020 Peer AS: 65020
                Age: 6:54       Metric2: 16777224 
                Validation State: unverified 
                Task: BGP_65020.10.0.0.0
                AS path: I  (Originator)
                Cluster list:  0.0.0.2
                Originator ID: 10.0.0.5
                Communities: target:10.0.0.1:0
                Import Accepted
                Localpref: 100
                Router ID: 10.0.0.0
                Secondary Tables: VRF-C2-HUB.mvpn.0
                Thread: junos-main 
                Indirect next hops: 1
                        Protocol next hop: 10.0.0.5 Metric: 16777224 ResolvState: Resolved
                        Indirect next hop: 0x2 no-forward INH Session ID: 0
                        Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                        Indirect path forwarding next hops: 2
                                Next hop type: Router
                                Next hop: 10.200.0.1 via ae0.0 weight 0x1
                                Session Id: 0
                                Next hop: 10.200.0.5 via ge-0/0/3.0 weight 0xf000
                                Session Id: 0
                                10.0.0.5/32 Originating RIB: inet.3
                                  Metric: 16777224 Node path count: 1
                                  Indirect next hops: 1
                                Protocol next hop: 10108 Metric: 16777219 ResolvState: Resolved
                                Inode flags: 0x202284 path flags: 0x0
                                Path fnh link: 0x1c93eee0 path inh link: 0x8449800
                                Label operation: Push 10106, Push 10107(top)
                                Label TTL action: no-prop-ttl, no-prop-ttl(top)
                                Load balance label: Label 10106: None; Label 10107: None; 
                                Indirect next hop: 0x80e8fd0 - INH Session ID: 0 Weight 0x1
                                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                                Indirect path forwarding next hops: 2
                                        Next hop type: Router
                                        Next hop: 10.200.0.1 via ae0.0 weight 0x1
                                        Session Id: 0
                                        Next hop: 10.200.0.5 via ge-0/0/3.0 weight 0xf000
                                        Session Id: 0
                                        10108 /52 Originating RIB: mpls.0
                                          Metric: 16777219 Node path count: 1
                                          Forwarding nexthops: 2
                                                Next hop type: Router
                                                Next hop: 10.200.0.1 via ae0.0 weight 0x1
                                                Session Id: 0
                                                Next hop: 10.200.0.5 via ge-0/0/3.0 weight 0xf000
                                                Session Id: 0

4:3:10.0.0.1:200:32:10.2.1.2:32:239.0.0.1:10.0.0.1:10.0.0.7/240 (1 entry, 0 announced)
        *BGP    Preference: 170/-101
                Next hop type: Indirect, Next hop index: 0
                Address: 0x1cce5814
                Next-hop reference count: 4
                Kernel Table Id: 0
                Source: 10.0.0.0
                Protocol next hop: 10.0.0.7
                Indirect next hop: 0x2 no-forward INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Active Int Ext>
                Local AS: 65020 Peer AS: 65020
                Age: 6:54       Metric2: 16777219 
                Validation State: unverified 
                Task: BGP_65020.10.0.0.0
                AS path: I  (Originator)
                Cluster list:  0.0.0.1
                Originator ID: 10.0.0.7
                Communities: target:10.0.0.1:0
                Import Accepted
                Localpref: 100
                Router ID: 10.0.0.0
                Secondary Tables: VRF-C2-HUB.mvpn.0
                Thread: junos-main 
                Indirect next hops: 1
                        Protocol next hop: 10.0.0.7 Metric: 16777219 ResolvState: Resolved
                        Indirect next hop: 0x2 no-forward INH Session ID: 0
                        Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                        Indirect path forwarding next hops: 1
                                Next hop type: Router
                                Next hop: 10.200.0.1 via ae0.0 weight 0x1
                                Session Id: 0
                                10.0.0.7/32 Originating RIB: inet.3
                                  Metric: 16777219 Node path count: 1
                                  Forwarding nexthops: 1
                                        Next hop type: Router
                                        Next hop: 10.200.0.1 via ae0.0 weight 0x1
                                        Session Id: 0
```
Here, we can see all the receivers!!! Now, our selective tree is created with success. 

The selective PMSI LSP is created:
```
root@R1> show mpls lsp 
Ingress LSP: 9 sessions
To              From            State Rt P     ActivePath       LSPname
10.0.0.2        10.0.0.1        Dn     0       -                10.0.0.2:10.0.0.1:200:mvpn:VRF-C2-HUB
10.0.0.4        10.0.0.1        Up     0 *                      10.0.0.4:10.0.0.1:200:mv1:VRF-C2-HUB
10.0.0.4        10.0.0.1        Up     0 *                      10.0.0.4:10.0.0.1:200:mvpn:VRF-C2-HUB
10.0.0.5        10.0.0.1        Up     0 *                      10.0.0.5:10.0.0.1:200:mv1:VRF-C2-HUB
10.0.0.5        10.0.0.1        Up     0 *                      10.0.0.5:10.0.0.1:200:mvpn:VRF-C2-HUB
10.0.0.7        10.0.0.1        Up     0 *                      10.0.0.7:10.0.0.1:200:mv1:VRF-C2-HUB
10.0.0.7        10.0.0.1        Up     0 *                      10.0.0.7:10.0.0.1:200:mvpn:VRF-C2-HUB
10.0.0.6        10.0.0.1        Up     0 *     primary          R1-R6-A
10.0.0.8        10.0.0.1        Up     0 *     primary          R1-R8-A
```

Now, let's check the traffic into CEs: 
```
[admin@CE2-3] > interface/monitor-traffic aggregate 
        rx-packets-per-second:         82
           rx-bits-per-second:  993.1kbps
     fp-rx-packets-per-second:          0
        fp-rx-bits-per-second:       0bps
          rx-drops-per-second:          0
         rx-errors-per-second:          0
        tx-packets-per-second:          1
           tx-bits-per-second:    2.7kbps
     fp-tx-packets-per-second:          0
        fp-tx-bits-per-second:       0bps
          tx-drops-per-second:          0
    tx-queue-drops-per-second:          0
         tx-errors-per-second:          0
....
[admin@CE2-4] > interface/monitor-traffic aggregate 
        rx-packets-per-second:         81
           rx-bits-per-second:  981.0kbps
     fp-rx-packets-per-second:          0
        fp-rx-bits-per-second:       0bps
          rx-drops-per-second:          0
         rx-errors-per-second:          0
        tx-packets-per-second:          0
           tx-bits-per-second:       0bps
     fp-tx-packets-per-second:          0
        fp-tx-bits-per-second:       0bps
          tx-drops-per-second:          0
    tx-queue-drops-per-second:          0
         tx-errors-per-second:          0
....
[admin@CE2-5] > interface/monitor-traffic aggregate 
        rx-packets-per-second:         83
           rx-bits-per-second:  982.2kbps
     fp-rx-packets-per-second:          0
        fp-rx-bits-per-second:       0bps
          rx-drops-per-second:          0
         rx-errors-per-second:          0
        tx-packets-per-second:          1
           tx-bits-per-second:     528bps
     fp-tx-packets-per-second:          0
        fp-tx-bits-per-second:       0bps
          tx-drops-per-second:          0
    tx-queue-drops-per-second:          0
         tx-errors-per-second:          0
```

I configured the MVPN before configure the PIM in all our network to show you the scale of the MVPN. We don't need the PIM running in all the network, we can run PIM only in the PE-CE interfaces, and the BGP scales perfectly! 

