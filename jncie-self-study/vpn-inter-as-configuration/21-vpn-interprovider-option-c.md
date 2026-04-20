# VPN Inter-Provider Option C Configuration

Hello friends, this is the last VPN that we'll configure in our JNCIE-SP journey. It's a bit sad, but at the same time, I'm happy we've explored so many features along the way.

Here is the topology we'll work with today:
<img width="1330" height="913" alt="image" src="https://github.com/user-attachments/assets/55aa7b65-0984-4cff-a42e-ae5fd49cf7c4" />

Our goal is to deliver the VPLS service to Customer 5, connected at P3. Following the same lore as the previous article, we acquired P3, and now we need to interconnect the services of our common customers.

This time, we'll use the VPN Inter-AS Option C. In other words, we'll use BGP-LU to advertise our loopbacks with labels to P3, and P3 will advertise its own loopback with labels to us. Then, we'll interconnect P3 with our network.

Do you get the logic? When P3 wants to forward traffic to R1, for example, it will use the label that we advertised through BGP. Then, R3 will receive the packet with this label, swap it to another label, and forward the traffic into our network. The reverse happens too: when R1 wants to forward traffic to P3, it pushes the label received from the BGP-LU route, and pushes another one to reach R3. Thanks to PHP on the network, R3 will receive the packet with only the BGP-LU label, swap this label, and forward it to P3.

Let's configure this now. First, let's enable the family inet labeled-unicast on our BGP mesh:
RR:
```
set protocols bgp group iBGP-AS65020-West family inet labeled-unicast rib inet.3
set protocols bgp group iBGP-AS65020-East family inet labeled-unicast rib inet.3
```
PEs:
```
set protocols bgp group iBGP-AS65020-East family inet labeled-unicast rib inet.3
```

Now that we have the family active on our backbone, we can establish a BGP-LU peering with P3. Next, we need to advertise the loopback routes of our network to P3, so we just need to create another term to do this.
```
set protocols bgp group eBGP-AS65503-Provider3 family inet labeled-unicast rib inet.3

set policy-options policy-statement Saida-P3 term from-igp from route-filter 10.0.0.0/24 upto /32
set policy-options policy-statement Saida-P3 term from-igp then accept
```
To advertise R3's loopback, we just need to include this route in inet.3. We can do this with a rib-group:
```
set routing-options rib-groups inet0-to-inet3 import-rib inet.0
set routing-options rib-groups inet0-to-inet3 import-rib inet.3

set routing-options interface-routes rib-group inet inet0-to-inet3
```
With this, we'll export all the routes present in inet.3:
```
root@R3> show route table inet.3 10.0.0.0/24 active-path

inet.3: 72 destinations, 133 routes (72 active, 0 holddown, 40 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/32        *[IS-IS/15] 5w0d 03:11:50, metric 16777224
                    >  to 10.200.0.6 via ge-0/0/3.0
10.0.0.1/32        *[LDP/9] 3w0d 03:11:02, metric 15
                    >  to 10.200.0.6 via ge-0/0/3.0, Push 33
                       to 10.200.0.11 via ge-0/0/4.0, Push 33
10.0.0.2/32        *[LDP/9] 3w0d 03:10:55, metric 10
                    >  to 10.200.0.6 via ge-0/0/3.0, Push 0
                       to 10.200.0.11 via ge-0/0/4.0, Push 35
10.0.0.3/32        *[Direct/0] 5w5d 23:51:38
                    >  via lo0.0
10.0.0.4/32        *[RSVP/7/1] 3w0d 03:10:06, metric 10
                    >  to 10.200.0.13 via ge-0/0/2.0, label-switched-path to-R4
10.0.0.5/32        *[LDP/9] 3w0d 03:11:02, metric 16777219
                    >  to 10.200.0.11 via ge-0/0/4.0, label-switched-path R3-R6-A
                       to 10.200.0.6 via ge-0/0/3.0, label-switched-path Bypass->10.200.0.11
10.0.0.6/32        *[RSVP/7/1] 5w0d 03:11:33, metric 16777214
                    >  to 10.200.0.11 via ge-0/0/4.0, label-switched-path R3-R6-A
                       to 10.200.0.6 via ge-0/0/3.0, label-switched-path Bypass->10.200.0.11
10.0.0.7/32        *[LDP/9] 3w0d 03:10:55, metric 16777224
                    >  to 10.200.0.11 via ge-0/0/4.0, label-switched-path R3-R6-A
                       to 10.200.0.6 via ge-0/0/3.0, label-switched-path Bypass->10.200.0.11
10.0.0.8/32        *[RSVP/7/1] 3w0d 03:10:45, metric 16777229
                    >  to 10.200.0.13 via ge-0/0/2.0, label-switched-path R3-R8-B
                       to 10.200.0.6 via ge-0/0/3.0, label-switched-path R3-R8-A
                       to 10.200.0.11 via ge-0/0/4.0, label-switched-path Bypass->10.200.0.6->10.200.0.0
```
Let's check the advertisements now:
```
root@R3> show route advertising-protocol bgp 172.16.3.6 table inet.3

inet.3: 73 destinations, 134 routes (73 active, 0 holddown, 40 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.0.0.0/32             Self                 16777224           I
* 10.0.0.1/32             Self                 15                 I
* 10.0.0.2/32             Self                 10                 I
* 10.0.0.3/32             Self                                    I
* 10.0.0.4/32             Self                 10                 I
* 10.0.0.5/32             Self                 16777219           I
* 10.0.0.6/32             Self                 16777214           I
* 10.0.0.7/32             Self                 16777224           I
* 10.0.0.8/32             Self                 16777229           I

root@R3> show route receive-protocol bgp 172.16.3.6 table inet.3

inet.3: 73 destinations, 134 routes (73 active, 0 holddown, 40 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.0.1.3/32             172.16.3.6                              65503 I
```
Everything looks good so far. Now, we need to configure the service.

P3 will establish a session with our RR and advertise the l2vpn routes to interconnect the sites. Let's configure this session and check what RT is being advertised.
```
set protocols bgp group eBGP-AS65503-LU type external
set protocols bgp group eBGP-AS65503-LU description eBGP-AS65503-LU
set protocols bgp group eBGP-AS65503-LU multihop ttl 24
set protocols bgp group eBGP-AS65503-LU local-address 10.0.0.0
set protocols bgp group eBGP-AS65503-LU peer-as 65503
set protocols bgp group eBGP-AS65503-LU neighbor 10.0.1.3 family l2vpn signaling
```
Basically, this is a multihop eBGP session that speaks the l2vpn family.

For this session to establish, we need to change some configurations on our RR:
```
delete group iBGP-AS65020-West family inet unicast nexthop-resolution
delete group iBGP-AS65020-West family inet unicast no-install
delete group iBGP-AS65020-East family inet unicast nexthop-resolution
delete group iBGP-AS65020-East family inet unicast no-install

set routing-options resolution rib inet.3 resolution-ribs inet.0
```
Without installing the route to P3, the RR can't establish the connection. In eBGP cases, Junos uses inet.0 to speak with the neighbor, and we weren't installing the route in inet.0. Finally, to resolve the BGP-LU route, we need to set the resolution RIB to resolve inet.3 routes using the inet.0 table. Now, our router installs the route and can establish the session.
```
root@RR> show route 10.0.1.3

inet.0: 216 destinations, 452 routes (216 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.1.3/32        *[BGP/170] 00:03:19, localpref 100, from 10.0.0.3
                      AS path: 65503 I, validation-state: unverified
                    >  to 10.200.0.26 via ge-0/0/1.0

inet.3: 64 destinations, 71 routes (64 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.1.3/32        *[BGP/170] 00:01:59, localpref 100, from 10.0.0.3
                      AS path: 65503 I, validation-state: unverified
                    >  to 10.200.0.26 via ge-0/0/1.0, Push 953

root@RR> show bgp summary group eBGP-AS65503-LU
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 3 Peers: 9 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.rtarget.0
                      44         21          0          0          0          0
inet.0
                     389        153          0          0          0          0
inet6.0
                      61         25          0          0          0          0
bgp.l3vpn.0
                      66         66          0          0          0          0
bgp.l3vpn-inet6.0
                       6          6          0          0          0          0
bgp.mvpn.0
                       5          5          0          0          0          0
bgp.l2vpn.0
                      15         15          0          0          0          0
bgp.evpn.0
                      45         45          0          0          0          0
inet.3
                      71         64          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.1.3              65503         19         23       0       0        6:38 Establ
  bgp.l2vpn.0: 1/1/1/0
```
Now, let's look at what routes we are receiving:
```
root@RR> show route receive-protocol bgp 10.0.1.3 table bgp.l2vpn.0 detail

bgp.l2vpn.0: 15 destinations, 15 routes (15 active, 0 holddown, 0 hidden)
*  10.0.1.3:500:7:1/96 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 10.0.1.3:500
     Label-base: 262145, range: 8, offset: 1
     Nexthop: 10.0.1.3
     AS path: 65503 I
     Communities: target:65020:1555 Layer2-info: encaps: VPLS, control flags:[0x0] , mtu: 0, site preference: 100
```
Now we know what RT is being used on the P3 network. We need to apply a similar logic to what we did previously on the L3VPN; we need to normalize this RT to interconnect the sites:
```
set policy-options community l2vpn-c5 members target:65020:500
set policy-options community l2vpn-c5-remote members target:65020:1555

set policy-options policy-statement Entrada-P3-L2VPN term 1 from community l2vpn-c5-remote
set policy-options policy-statement Entrada-P3-L2VPN term 1 then community add l2vpn-c5
set policy-options policy-statement Entrada-P3-L2VPN term 1 then community delete l2vpn-c5-remote
set policy-options policy-statement Entrada-P3-L2VPN term 1 then accept
set policy-options policy-statement Entrada-P3-L2VPN then reject

set policy-options policy-statement Saida-P3-L2VPN term 1 from community l2vpn-c5
set policy-options policy-statement Saida-P3-L2VPN term 1 then community add l2vpn-c5-remote
set policy-options policy-statement Saida-P3-L2VPN term 1 then community delete l2vpn-c5
set policy-options policy-statement Saida-P3-L2VPN term 1 then accept
set policy-options policy-statement Saida-P3-L2VPN then reject

set protocols bgp group eBGP-AS65503-LU import Entrada-P3-L2VPN
set protocols bgp group eBGP-AS65503-LU export Saida-P3-L2VPN
```
With this, we'll receive the l2vpn routes and change the RT before advertising them to our PEs, and vice-versa. Now, let's check the advertisements:
```
root@RR> show route receive-protocol bgp 10.0.1.3

bgp.l2vpn.0: 16 destinations, 16 routes (16 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  10.0.1.3:500:7:1/96
*                         10.0.1.3                                65503 I
  10.0.1.3:500:7:9/96
*                         10.0.1.3                                65503 I

root@RR> show route advertising-protocol bgp 10.0.1.3

bgp.l2vpn.0: 16 destinations, 16 routes (16 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  10.0.0.2:500:4:1/96
*                         Not advertised                          I
  10.0.0.2:500:4:9/96
*                         Not advertised                          I
  10.0.0.3:500:5:1/96
*                         Not advertised                          I
  10.0.0.3:500:10:1/96
*                         Not advertised                          I
  10.0.0.3:500:10:9/96
*                         Not advertised                          I
  10.0.0.4:500:5:1/96
*                         Not advertised                          I
  10.0.0.4:500:5:9/96
*                         Not advertised                          I
  10.0.0.5:500:6:1/96
*                         Not advertised                          I
  10.0.0.5:500:6:9/96
*                         Not advertised                          I
```
Here we have a specific problem: the l2vpn routes aren't being advertised to P3. This happens because, in eBGP sessions, the router changes the next-hop attribute by default.

If we check the state of the pseudowire on R2, we can see that it's up:
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
    5                         rmt   Up     Apr 16 15:17:08 2026           1
      Remote PE: 10.0.0.3, Negotiated control-word: No
      Incoming label: 28, Outgoing label: 294
      Local interface: vt-0/0/0.1048590, Status: Up, Encapsulation: VPLS
        Description: Intf - vpls VPLS-CE5 local site 4 remote site 5
      Flow Label Transmit: No, Flow Label Receive: No
    6                         rmt   Up     Apr 16 15:17:08 2026           1
      Remote PE: 10.0.0.5, Negotiated control-word: No
      Incoming label: 29, Outgoing label: 22
      Local interface: vt-0/0/0.1048589, Status: Up, Encapsulation: VPLS
        Description: Intf - vpls VPLS-CE5 local site 4 remote site 6
      Flow Label Transmit: No, Flow Label Receive: No
    7                         rmt   Up     Apr 16 15:38:52 2026           1
      Remote PE: 10.0.1.3, Negotiated control-word: No
      Incoming label: 30, Outgoing label: 262148
      Local interface: vt-0/0/0.1048591, Status: Up, Encapsulation: VPLS
        Description: Intf - vpls VPLS-CE5 local site 4 remote site 7
      Flow Label Transmit: No, Flow Label Receive: No
    10                        rmt   RM
```
But why? Because our RR advertised the route without changing the next-hop, following the standard rules of BGP. Then, R2 receives the route and advertises its own route, so it considers the pseudowire up.

Fortunately, there is a knob to change this behavior:
```
set protocols bgp group eBGP-AS65503-LU multihop no-nexthop-change
```
This way, when our RR advertises the routes, the next-hop will not be changed.
```
root@RR> show route advertising-protocol bgp 10.0.1.3

bgp.l2vpn.0: 16 destinations, 16 routes (16 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  10.0.0.2:500:4:1/96
*                         10.0.0.2                                I
  10.0.0.2:500:4:9/96
*                         10.0.0.2                                I
  10.0.0.3:500:5:1/96
*                         10.0.0.3                                I
  10.0.0.3:500:10:1/96
*                         10.0.0.3                                I
  10.0.0.3:500:10:9/96
*                         10.0.0.3                                I
  10.0.0.4:500:5:1/96
*                         10.0.0.4                                I
  10.0.0.4:500:5:9/96
*                         10.0.0.4                                I
  10.0.0.5:500:6:1/96
*                         10.0.0.5                                I
  10.0.0.5:500:6:9/96
*                         10.0.0.5                                I

root@P3-1> show vpls connections
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
VM -- VLAN ID mismatch

Legend for interface status
Up -- operational
Dn -- down

Instance: VPLS-CE5
  Local site: s7 (7)
    connection-site           Type  St     Time last up          # Up trans
    4                         rmt   Up     Apr 15 15:53:21 2026           1
      Remote PE: 10.0.0.2, Negotiated control-word: No
      Incoming label: 262148, Outgoing label: 30
      Local interface: lsi.1048833, Status: Up, Encapsulation: VPLS
        Description: Intf - vpls VPLS-CE5 local site 7 remote site 4
    5                         rmt   Up     Apr 15 15:53:21 2026           1
      Remote PE: 10.0.0.3, Negotiated control-word: No
      Incoming label: 262149, Outgoing label: 297
      Local interface: lsi.1048832, Status: Up, Encapsulation: VPLS
        Description: Intf - vpls VPLS-CE5 local site 7 remote site 5
    6                         rmt   Up     Apr 15 15:53:21 2026           1
      Remote PE: 10.0.0.5, Negotiated control-word: No
      Incoming label: 262150, Outgoing label: 25
      Local interface: lsi.1048834, Status: Up, Encapsulation: VPLS
        Description: Intf - vpls VPLS-CE5 local site 7 remote site 6
    10                        rmt   RM
```
Now, the pseudowires are completely established. Let's ask the customer to test the connectivity:
```
[admin@CE5-7] > ping 172.50.1.4
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.50.1.4                                 56  64 10ms716us
    1 172.50.1.4                                 56  64 4ms791us
    sent=2 received=2 packet-loss=0% min-rtt=4ms791us avg-rtt=7ms753us max-rtt=10ms716us

[admin@CE5-7] > ping 172.50.1.5
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.50.1.5                                 56  64 13ms91us
    1 172.50.1.5                                 56  64 3ms654us
    sent=2 received=2 packet-loss=0% min-rtt=3ms654us avg-rtt=8ms372us max-rtt=13ms91us

[admin@CE5-7] > ping 172.50.1.5
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.50.1.5                                 56  64 5ms10us
    1 172.50.1.5                                 56  64 3ms363us
    sent=2 received=2 packet-loss=0% min-rtt=3ms363us avg-rtt=4ms186us max-rtt=5ms10us

[admin@CE5-7] > ping 172.50.1.6
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.50.1.6                                 56  64 38ms639us
    1 172.50.1.6                                 56  64 12ms881us
    sent=2 received=2 packet-loss=0% min-rtt=12ms881us avg-rtt=25ms760us max-rtt=38ms639us
```
Everything is great!!!!

With this, we've finished our VPN journey, and we can move on to Class of Service. See you next time, bye!
