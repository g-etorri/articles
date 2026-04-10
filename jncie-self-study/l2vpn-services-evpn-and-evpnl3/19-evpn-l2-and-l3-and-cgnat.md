# EVPN Layer 2 and Layer 3 Configuration

Hello guys, how about you? Everything is going right? I expect you are fine! 

Today, we'll dive in a fun topic that I fall in love in the first day, EVPN! 

EVPN is basically the natural evolution of L2VPNs, I'm saying L2VPN in general because we have some flavor of EVPN, that here I won't explore. And we can integrate L2 EVPNs into L3VPN, that we can call L3 EVPNs! 

Here is the topology that we'll use today:
<img width="1507" height="1138" alt="image" src="https://github.com/user-attachments/assets/516631b6-f67d-442b-a514-99307dff7a3a" />

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


**Ethernet AD Per-EVI** is created for each EVI in the ESI. This route have two key functions, enabling the load-balance in all-active mode, and another one enabling the backup-paths in the single-active mode. 

In the output, first we have the AD Per-ESI, and so on the AD Per-EVI. 
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
EVPN routes type 3: Called Inclusive Multicast Ethernet Tag Route, this route is used to build a inclusive tree to forward not only multicast traffic, but broadcast and unknown unicast also, like a MVPN, indeed, this technique was copied of MVPNs! Hahahah, nothing is created, everything is tranformed. In the route below we can see almost all characteristics that we have in route type 2, but we have the router-id of the remote PE instead of MAC/IP. In the details of the route we can see the PMSI and the label that will be used to forward the BUM traffic, this is the most important information. 
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

EVPN routes type 4: Called Ethernet Segment Routes, this route is used to PEs discover the other PEs connected in the same ESI and to avoid loops in one way, I'm saying in one way because we have two kinds of loop that can happen here, the loop where the BUM traffic is forwarded by a remote CE, and the loop where the BUM traffic was forwarded by the local CE. The route type 4 prevent the loop where the BUM traffic is forwarded into EVPN by a remote CE, this route is used to have a DF election, where the two PEs will decide what PE can forward the BUM traffic, yes, only one PE can forward the BUM traffic trough the ESI, preventing loops. This election is exclusive of an ESI, if the PE have other CEs in the same EVI, it can forward the BUM traffic for them normally, the process of election happens trough a calculation defined in the RFC 7432, and it's not important right now, but with this both PEs will have the same result, and only one will be the DF. 
The route format is ```ROUTE-TYPE:RD::ESI:PE``` and basically, this route is used mainly to discover the PEs in the same ESI. This is the content of the route after all. R1 receives the route from R2 with the same ESI, and now it knows that R2 is connected into ESI 01, then the DF election happens. 
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
We can see who is the DF of the ESI on the instance output, with some details of when was the last DF election: 
```
root@R1> show evpn instance extensive
Instance: EVPN-CE12
....
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
 root@R3> show evpn instance extensive
 Instance: EVPN-CE34
....
  Number of ethernet segments: 2
    ESI: 00:00:00:00:00:00:00:00:00:02
      Status: Resolved by IFL ae1.3400
      State-Bitfield: 0x43
      ESI Route Label: 21
      ESI Refcount: 1
      ESI Num Macs: 1, ESI Num SGDBs: 0
      Token-Route NH: 0
      Number of Local interfaces: 1
      Local interface: ae1.3400, Status: Up/Forwarding
      Number of remote PEs connected: 1
        Remote-PE        MAC-label  Aliasing-label  Mode
        10.0.0.4         22         22              all-active
      DF Election Algorithm: MOD based
      Designated forwarder: 10.0.0.3
      Backup forwarder: 10.0.0.4
      Last designated forwarder update: Apr 10 12:40:51
      Advertised MAC label: 22
      Advertised aliasing label: 22
      Advertised split horizon label: 23
 ....
```
