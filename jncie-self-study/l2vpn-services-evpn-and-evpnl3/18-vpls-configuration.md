# VPLS Configuration

Hello guys, I expect everyone is fine today! Here I am again, to bring to you another article, now about VPLSs!!! 

The topology that we'll use today is here:
<img width="1928" height="1148" alt="image" src="https://github.com/user-attachments/assets/52d5e7af-1bef-48f5-ac4f-1d18b949e8e9" />
You can notice that we have two multi-homed sites, this is to explain to you how do this. We can configure multihomed sites in VPLS signalled via LDP or BGP. 

First, let's clarify the configuration with this table:
| Customer | Site | Router | Protocol | PE-CE Interface | VLAN | PE |
| ------- | ---- | ------ | ----------- | ------------------- | -------- | ------ |
| C5      | S4   | CE5-4  | BGP         | ge-0/0/8            | 500, 501 | R2     |
| C5      | S5   | CE5-5  | BGP         | ge-0/0/8, ge-0/0/7  | 500, 501 | R3, R4 |
| C5      | S6   | CE5-6  | BGP         | ge-0/0/9            | 500, 501 | R5     |
| C6      | S1   | CE6-1  | LDP         | ge-0/0/9            | 600, 601 | R1     |
| C6      | S2   | CE6-2  | LDP         | ge-0/0/5, ge-0/0/8  | 600, 601 | R8, R7 |
| C6      | S3   | CE6-3  | LDP         | ge-0/0/6            | 600, 601 | R6     |

First, let's configure the Customer 5 VPLS, the Kompella VPLS. This time we don't need to configure anything on the backbone, the VPLS will use the l2vpn family to exchange the information. 

Another detail that we have here, in each connection we have two VLANs, to mantain two different broadcast domain, let's made a VPLS in VLAN-AWARE mode that separate each broadcast domain of the VPLS. Let's go. 

Starting on R2, the configuration is:
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
Note: The command ```vlan-id all``` specify the VPLS in vlan-aware mode. Here we have similar elements that we have in the VPWS Kompella. The ```no-tunnel-services``` knob creates a LSI/VT interface, avoiding use the lt interfaces. 

We can apply the same logic in the other PEs of the network. To avoid layer 2 loops, the configuration of the multihomed sites needs to be specials. Let's check the VPLS connections now, then configure the multihomed site to give more attention.
In the BGP, we'll receive the route from Site 6:
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
Here we can see the label-base, the range and the offset, this is used to know what label we need to use to forward the traffic to this site. With this, the Junos automatically create the VPLS connection:
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
We can see the flags that Junos can set if the circuit has some problem. Like the L2CKT flags that I show in the previous article. Now, the main difference between VPLS and VPWS, in the VPWS we don't have mac learning and we can't have more than one peer, logically. With VPLS, we have the mac learning enabled, this way we can differenciate the source of the MAC address, and with this, we can have multiple peers. Let's check the mac-table of the VPLS:
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
We can see here the vt interface, that is basically an virtual-interface that represents the push of the VPLS label. And other important thing, the two different mac-tables! Here we can confirm that our vlan-aware mode is working perfectly!  

Before ask the customer to give us some tests, let's finish the service configuration with the multihomed site. Let's go! 

If you remember the JNCIP-SP studies, in the VPLS Kompella the multihomed is not active-active, but active-standby, only one PE will be the traffic forwarder to the customer, and this is defined by the local-preference value of the routes. Let's see it. 

In R3, all the logic of the VPLS configuration is the same, but we are including the ```multi-homing``` and the ```site-preference``` knob into the VPLS configuration:
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
In R4, we'll do the same, but changing the ```site-preference``` to backup:
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
Now, let's check the routes: 
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
Note the local-preference that is the site preference, this way the R2 can know what is the PE designated as forwarder. Now, let's check the connections on the R3 and R4! 

In R3, we can note that we have the other two sites at UP State, but we can note that we have the Site 5 also. This happens because R3 is receiving the route of R4 that have the Site 5, and the flag is RN, that is Remote Site Not Designed, basically, the R4 is not the traffic forwarder of this site. 
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
Now, let's check the R4: 
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
The connection with all the sites are with the flag LN, Local Site Not Designed, that is, R4 is not the traffic forwarder of the Site 5. This configuration ensures that we don't have any layer 2 loop here caused by the multihomed connection. 

Let's check the final mac-table and ask the customer to send us some tests: 
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
Here we have the Site 5 with connection with the other sites in the network:
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
And, mission accomplished here! Customer 5 with an excellent service. 



INTERLIGAR O L2CKT DO CLIENTE C5 com o VPLS
