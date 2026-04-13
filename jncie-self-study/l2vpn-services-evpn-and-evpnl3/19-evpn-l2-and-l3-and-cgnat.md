# EVPN Layer 2 and Layer 3 Configuration

Hello guys, how about you? Everything is going right? I expect you are fine! 

Today, we'll dive in a fun topic that I fall in love in the first day, EVPN! 

EVPN is basically the natural evolution of L2VPNs, I'm saying L2VPN in general because we have some flavor of EVPN, that here I won't explore. And we can integrate L2 EVPNs into VRFs, that we can call L3 EVPNs! 

Here is the topology that we'll use today:
<img width="1507" height="1138" alt="image" src="https://github.com/user-attachments/assets/516631b6-f67d-442b-a514-99307dff7a3a" />

The situation is as follows, our customer wants to have connection layer 2 and layer 3 connection between the sites. And also wants to have connection to the internet.
First, we'll configure a normal EVPN, and to provide internet access, we'll integrate the EVPN into a VRF. All right, let's go. 

First things first, here is the table to follow to configure the EVPN: 
| CE Router | CE's IP Address | PE | PE-CE Interface | LACP  | VLAN ID | Default Gateway |
| ----------- | ----------------- | ----------- | ------------------- | ----- | ------- | ---------------- |
| CE12-1      | 10.16.12.12/24   | R1          | ae1.1200            | Active | 1200    | 10.16.12.254/24 |
| CE12-1      | 10.16.12.12/24   | R2          | ae1.1200            | Active | 1200    | 10.16.12.254/24 |
| CE34-1      | 10.16.34.34/24   | R3          | ae1.3400            | Active | 3400    | 10.16.34.254/24 |
| CE34-1      | 10.16.34.34/24   | R4          | ae1.3400            | Active | 3400    | 10.16.34.254/24 |

Let's enable the evpn family in our BGP mesh: 
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
Ok, with the family EVPN running in the network, we can configure the EVI! 

Both sites belongs the same customer, and both sites are multihomed. Different than VPLS, here we can do a active-active multihoming, using ESI-LAGs! 
What is this? Basically, we'll configure a LACP in both PEs with the same system-id, for the customer the PEs will be like a single one, the traffic will be load-balanced between the two PEs. And the traffic for the CE, will be load-balanced using the aliasing label! To avoid loops, we have a DF Router, that will forward the BUM traffic. This is perfect, I confess. 

Another detail in the topology, is the VLAN used, to permit the communication between the two sites, we'll do a VLAN normalization, swapping the VLAN tag to 1234. 

Let's start with the configuration in R1 and R2:
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
First, we'll bind the physical interface into the LAG, and in the LAG we need to define the ESI and this mode of operation, that will be active-active. The LACP system-id needs to be the same in both PEs.

In the instance, we need to create an EVPN instance, and the encapsulation will be MPLS, you don't need to specify the encapsulation, by default is MPLS. And the EVI needs RD and RT, like a L3VPN! All the components makes EVPN fantastic and scalable. You can note the ```vlan-id``` knob, this is used to do the VLAN normalization. 

Now, let's make the configuration of the other site to start some tests. We'll apply the same logic:
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
With this, we can learn the MAC addresses from PE-CE interface, but we don't have any connectivity between the sites yet, do you know why? Because the IP addresses don't belongs to the same network. We only have layer 2 connectivity, let's see the LLDP neighbors...
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
Both CEs have LLDP neighboring, but don't have any layer 3 connectivity. 

Let's check the EVPN routes, to see what we already have. 
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
Here we have routes type 1, 2 and 3. 
EVPN routes type 1: Here we have two sub-types of route, hahaha. We can define two kinds of route type 1. 

The Ethernet Auto-Discovery Per-ESI and Per-EVI. **Ethernet AD Per-ESI** is created for the entire ESI, if the PE is connected in two ESIs, it will generate two routes, this route contains each RT of EVI in the ESI, in other words, if in one ESI we have 10 EVIs, this route will contain the 10 RTs of the EVIs, also will contain a special community that is a special ESI label (This is the main difference between the routes, the Per-EVI uses the MPLS label field of the NLRI) and the ESI number.

This route have two key functions, the first is the withdrawal of the routes, if the connection PE-CE fails in the ESI, this route type 1 is withdrawal by the PE, the remote PEs will converge this traffic to the other PE of the ESI, this is extremely useful, you can understand this better if we have 500 MACs learned by the interface that goes down, and if we have a lot of EVIs configured in the interface, with the withdrawal of the route type 1, the remote PEs can converge the traffic of all EVIs in the ESI, and don't need to wait the withdrawal of all 500 routes MAC/IP, got it? 

And the second function is protect avoid loops generated by the local CE, this is achieved trough the community of the route. In the community field we have two things, the first is a bit that identifies if the ESI is single-active or active-active, and the second is the special ESI label, now follow me in the tought, if CE12-1 sends a broadcast packet to R2, R2 will flood this traffic to R1 that is DF, then the traffic will be forwarded to CE12-1 and so on... the loop is happening. 

To avoid this, R2 could simply not forward the traffic to R1, but if R1 haves another site connected? We could had a problem. And thinking in this scenario, this special label is used, when R2 needs to forward a BUM traffic to another PE of the same ESI, the traffic will be forwarded with a new bottom label, this special ESI label. When R1 pops the label, will identify this label as a "I AM BUM TRAFFIC OF X ESI, PLEASE AVOID LOOPS", and with this, R1 simply forwards the traffic to the other sites, or discards the traffic, this label also is called as **SPLIT HORIZON LABEL**.   


**Ethernet AD Per-EVI** is created for each EVI in the ESI. This route have two key functions, enabling the load-balance in all-active mode, and another one enabling the backup-paths in the single-active mode. Let's go to the aliasing-label, follow me in the tought again. The CE12-1 sends a frame to the R1 but not to R2, naturally the next-hop is the ESI, then the remote PEs will have the ESI as next-hop and can send the traffic to R2. R2 can learn the MAC trough the EVPN advertised by R1, but what if R2 don't process this route in time, then R2 will flood this traffic naturally. 

To avoid this scenario, the remote PEs will use the aliasing-label instead of the normal label of EVPN, with this label the PE2 even without the MAC address learned, will forward the traffic to the ESI, making the load-balance of the traffic without loss any packet. 

In the backup-paths process that happens when we have a single-active ESI, the remote PEs will install the routes for the secondary PEs as backup in the FIB, then, when the remote PEs receive the withdrawal of the AD Per-ESI, this backup routes will become active. 

You got the type 1 routes? Both kind of routes will work together, to discovery wath EVIs we have in the ESI and how forward the traffic in the right manner. 

In the output, first we have the AD Per-ESI with the ```ROUTE-TYPE:RD::ESI```, you can notice the esi-label community and the RT of EVI in the ESI, that identifies if we have an active-active or single-active ESI. And so on the AD Per-EVI with the ```ROUTE-TYPE:RD::ESI```, and we can notice the RT of the EVI and the aliasing label advertised.  
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
EVPN routes type 2: Called MAC/IP Routes, are responsible by advertise the MAC addresses, and also can advertise the MAC+IP addresses trough ARP snooping. This routes also contain the labels that the remote PE must use to forward the traffic, the VLAN/BRIDGE-DOMAIN that the MAC address belongs, and the ESI where the route was learned. 
The route structure is: ```ROUTE-TYPE:RD::VLAN::MAC-ADDRESS```
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
EVPN routes type 3: Called Inclusive Multicast Ethernet Tag Route, this route is used to build a inclusive tree to forward not only multicast traffic, but broadcast and unknown unicast also, like a MVPN, indeed, this technique was copied of MVPNs! Hahahah, nothing is created, everything is tranformed. 

In the route below we can see almost all characteristics that we have in route type 2, but we have the router-id of the remote PE instead of MAC/IP. In the details of the route we can see the PMSI and the label that will be used to forward the BUM traffic, this is the most important information. 
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

And, we have a hidden EVPN route present in our topology, that are route type 4!!! 

EVPN routes type 4: Called Ethernet Segment Routes, this route is used to PEs discover the other PEs connected in the same ESI and to avoid loops in one way, I'm saying in one way because we have two kinds of loop that can happen here, the loop where the BUM traffic is forwarded by a remote CE, and the loop where the BUM traffic was forwarded by the local CE. 

The route type 4 prevent the loop where the BUM traffic is forwarded into EVPN by a remote CE, this route is used to have a DF election, where the two PEs will decide what PE can forward the BUM traffic, yes, only one PE can forward the BUM traffic trough the ESI, preventing loops. This election is exclusive of an ESI and occurs for each EVI in the ESI, if the PE have other CEs in the same EVI, it can forward the BUM traffic for them normally, the process of election happens trough a calculation defined in the RFC 7432, and it's not important right now, but with this both PEs will have the same result, and only one will be the DF.

The route format is ```ROUTE-TYPE:RD::ESI:PE``` and basically, this route is used mainly to discover the PEs in the same ESI, or identify the ESI and his PEs in the network. This is the content of the route after all. R1 receives the route from R2 with the same ESI, and now it knows that R2 is connected into ESI 01, then the DF election happens. For the other PEs, they will know that R1 and R2 are connected into this ESI, indeed the MAC/IP routes have the ESI as next-hop. 
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


Now, verifying the EVI, we can see all a lot of details, the VLAN used, the MAC database stats, the number of bridge domains, the neigbors of EVI, the ESIs in the EVI and the PEs connected in the ESI also, DF PE, aliasing label, split-horizon label and so on...   
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
EVPN is the most complete solution that we have currently, and I love it. 

Now, let's adjust this connectivity, we'll use here the the EVPN Virtual Gateway Address, this way the CE can use any PE to have L3 connectivity in the network, and with this we can provide internet access to the EVPN customers also. The EVPN Virtual Gateway Address permit us to have the anycast gateway configured and also have another address to troubleshooting.  

First, let's provide communication between the sites. The configuration is very simple, create an IRB interface as a gateway and have an unique address inside the network also. Then add the routing-interface into EVPN and don't advertise this route with the default-gateway community.  
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

With this, we have the anycast gateways configured and let's check our routing-table: 
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
Now, we have the MAC and MAC/IP routes from both CEs and all PEs also. Everythng looks fine, let's ask the customer to check the connectivity between the sites:
```
[admin@CE34-1] > tool traceroute 10.16.12.12
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS      LOSS  SENT  LAST   AVG    BEST  WORST  STD-DEV
1  10.16.34.4   0%       4  4.2ms  107.3  4.2   393.4  165.4
2  10.200.0.2   0%       4  7.8ms  169.6  7.8   625.6  263.5
3  10.16.12.12  0%       4  2.3ms  10.3   2.3   21.3   7.1
```
And... Is ok, now, we need to provide the internet access to the customer.

First, let's create a VRF in R3 to start the L3-EVPN. So, we'll have two interface to the CGNAT, one we can consider as an inside interface, and the other one the outside interface, inside = LAN and outside = WAN, basically. 
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
Here, R3 will advertise an default route and will receive the public prefix of the CGNAT. CGNAT will advertise the default route to R3 again, but this time into the VRF, and R3 will advertise this into EVPN. You got it? Yeah, I know. 

All right, let's check the advertisements now:
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
Here, we are advertising the local route, of the IRB interface. If you are following the tought, you know here that we have a problem. 

Let's check the global RIB now:
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
Here, we are advertising the default route, that the CGN is advertsiing into the VRF, and we are receiving the public prefix, that we are marking the customer community to advertise to our peerings. 

Now, may are you asking yourself that you need to configure the VRF in the other PEs and all the things will be resolved. And you are right. But here the things are different, we won't use inet-vpn to exchange the routes, we'll use the evpn to do this. 

First, we need to configure the VRF on the other PEs:
```
set routing-instances VRF-EVPN instance-type vrf
set routing-instances VRF-EVPN description VRF-EVPN
set routing-instances VRF-EVPN interface irb.1234
set routing-instances VRF-EVPN route-distinguisher 10.0.0.1:12341
set routing-instances VRF-EVPN vrf-target target:65020:12341
set routing-instances VRF-EVPN vrf-table-label
```

To show you the difference between this VRF and the L3VPN VRF, when you want to see the routes from a VRF, if you not complete the command with inet.0 ou bgp.l3vpn.0, Junos shows both tables, the primary table that is bgp.l3vpn.0 and the secondary table, where the Junos put the inet routes correctly. 
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
In a L3-EVPN, we can exchange the routes trough the evpn family, and if we have the inet-vpn enable, we can exchange the routes trough this family also, check here: 
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
With the route type 5, the natural move is excluding the inet-vpn family in the backbone, to test the reliability, let's test the routing without inet-vpn configured. 
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
Here we can see an example of migration from L3VPN to L3-EVPN, the customer lost only a packet, and in the RIB we have the EVPN type 5 route installed. 

Now, let's test the internet access of the Site 12:
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
Perfectly trough EVPN only!!! 

If we want to optimize the routing, we can export the MAC-IP routes into VRF also. Follow my tought here, R1 and R2 have connectivity with CE12-1, both of them haves the 10.16.12.0/24 configured into IRB interface and exporting this prefix to R3:
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
If the connection between R1-CE fails, the IRB interface will not goes down, then R1 will export the prefix yet. To solve this possible issue, we can export the MAC-IP routes into VRF, to optimize this routing. 

We need to add a simple knob into EVPN instances:
```
set routing-instances EVPN-CE12 protocols evpn remote-ip-host-routes
set routing-instances EVPN-CE34 protocols evpn remote-ip-host-routes
```
This knob makes the PE leak the MAC-IP routes from evpn rib, to the inet rib. This knob not allow the PE to generate type 5 routes, only do a local process to put the mac-ip routes into the inet rib. 
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
Basically, we can route the traffic for a specific IP trough the L2-EVPN!!! 

All of our goals was accomplished today. I like so much write this text and explore the features of EVPN. I expect that you liked it also. 

See you soon in the inter-as VPNs!!! Bye. 
