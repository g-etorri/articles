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

Now, let's to the VPLS Martini! Here the configuration is simple and objective, like L2CKTs. 

I'm applying the configuration on R1, but we'll follow the same logic in the other PEs. Again, let the multihomed site without configuration to avoid loops.
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
Here we need to define the same parameters that we have in the L2CKT, encapsulation, MTU and vpls-id needs to match, if not the VPLS will not be UP. I added the flow-label into the circuit to improve the load-balance of the traffic, is only a good practice. In the same logic of the VPLS Kompella, we'll use the vlan-aware mode of the VPLS, mantaining the broadcast domains separated. 

Ok, with the configuration maded in the R1 and R6, let's check the outputs: 
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
Here we have the pseudowire to R6 established. Let's look at the LDP FEC 128 message now: 
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
In the LDP messages we can see the all the parameters, including MTU, flow-label bits, MTU, vpls-id/virtual-circuit-id and encapsulation. This is fantastic!!! 

Now, let's go to the multihomed site. 

To mantain only one traffic forwarder in the VPLS, we need to establish the pseudowire to one router. I choose the R8, so on the R1 and R6 let's add the configuration: 
```
set routing-instances VPLS-CE6 protocols vpls neighbor 10.0.0.8 revert-time 30
set routing-instances VPLS-CE6 protocols vpls neighbor 10.0.0.8 backup-neighbor 10.0.0.7 standby
```
This way, the pseudowire to R8 and R7 will be established, but the pseudowire for R7 will be used only if the R8 pseudowire fails. The revert-time will revert the traffic to use to R8 pseudowire again after it goes up in X seconds, here is defined in 30 seconds. 

In R8 and R7, let's configure the VPLS, but to avoid loops, we can't configure a pseudowire between then. 
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
With this, let's check the status on R1:
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
You can see here, R8's pseudowire is UP, and R7's pseudowire is with the ST flag, as Standby Connection. 

If you go look into R7, you can see the pseudowires established to R1 and R6. This is what the ```stanby``` knob do. The pseudowire is up but not in use. 
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
With this, theoritically the R7 will not learn any MAC address from remote PEs: 
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
Different than VPLS Kompella, we don't have any signalling between the multihomed PEs to identify a multihomed site, then the R7 is learning the MAC addresses of local-interface. 

Now, in R8 we have the mac-table complete: 
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
Ok, now let's apply some constraints here, I don't want that this customer have more than 20 MAC addresses on the mac-table, I'll limit this and drop the packets if the customer tresspass the limit: 

Let's apply this in all PEs:
```
set routing-instances VPLS-CE6 protocols vpls mac-table-size 20
set routing-instances VPLS-CE6 protocols vpls mac-table-size packet-action drop
set routing-instances VPLS-CE6 protocols vpls interface-mac-limit 20
```
In case of some problem in the customer LAN, we'll avoid some overload in our network. 

Let's ask the customer some tests to confirm if the service is complete: 
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
Everything looks good!!!

Now, the Customer 5 is calling me...

He wants to connect his Site 1, that haves VPWS into VPLS! Ok man, let's do it. 

At R7, let's add another VLAN to this connection, and create a new pseudowire into L2CKT: 
```
set interfaces ge-0/0/7 unit 500 description S1-VPLS
set interfaces ge-0/0/7 unit 500 encapsulation vlan-ccc
set interfaces ge-0/0/7 unit 500 vlan-id 500
set routing-instances L2VPN-CE5-S1 protocols l2vpn site s1 interface ge-0/0/7.500 remote-site-id 10
set routing-instances L2VPN-CE5-S1 interface ge-0/0/7.500
```
This way, we'll transport all the traffic of the VLAN 500 to the Site 10, Site 10 don't exist, is only a ficitcious site that we'll create on R3 to interconnect the services. 

In R3, we need to create two units on the logical-tunnel interface. An unit to include into VPWS, and another one to include into VPLS, both with the VLAN 500 configured, but with different encapsulation to work properly into VPLS and VPWS. 
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
Into L2VPN, let's add our Site 10 and the connection to the Site 1, and on the VPLS let's add another site including the lt interface also. 

Now, the traffic of the Site 1 that comes from VLAN 500, will go to the R3, and in R3 will be forwarded trough lt-0/0/0.0 to lt-0/0/0.1, then forwarded to the VPLS! And the opposite occurs too. You got it? It's easy, come on. 

Enjoying this new requirement, I'll add limits on the VPLS also. I

I'll limit the mac-learning of the VPLS in 16 addresses:
```
set routing-instances VPLS-CE5 protocols vpls mac-table-size 16
set routing-instances VPLS-CE5 protocols vpls interface-mac-limit 16
```
But this time, I won't drop the packets. 

Let's ask the customer if Site 1 have connection with the VPLS sites:
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
Everything is ok!!! 

All our goals is accomplished here, I will add a bonus here. Do you remember that I commented in the previous article about LDP FEC 128? VPLS Martini and L2CKT uses the same message, we can establish a pseudowire between two routers using VPLS at one side, and VPWS in the other side. I'll show you an example, we can define this as H-VPLS, like a hub-and spoke topology. 

At R1 we'll have a VPLS configured, and on R5 and R6 we'll have L2CKT. 
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

Now, on R1 we have the pseudowires established correctly:
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
This is naturally a hub-and-spoke topology, R5 and R6 can't communicate this way, we can add a mesh-group including the two pseudowires and enabling the local-switch to change the default behavior. 

That's all folks!!! The next time I'll write about EVPN, an exciting topic!!! See you next. 
