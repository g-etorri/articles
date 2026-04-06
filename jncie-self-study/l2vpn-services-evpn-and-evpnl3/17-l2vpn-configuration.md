# L2VPN Configuration

Hello guys, this time we'll configure the L2VPNs!!! In Junos, the VPWS was refered as L2VPN, despite VPLS is a L2VPN also. Today, we'll configure the VPWS connection in the two models, Kompella and Martini! 

The topology that we'll follow is here: 
<img width="987" height="821" alt="image" src="https://github.com/user-attachments/assets/f2aaab60-af94-4b3d-bb83-14d5921f2366" />

First of all, let's clarify all the circuits:
| Cliente | Site | Router | Sinalização | Interface para o CE | VLAN    | Conexão      | PE  |
| ------- | ---- | ------ | ----------- | ------------------- | ------- | ------------ | --- |
| C4      | S1   | CE4-1  | LDP         | ge-0/0/7            | 412,413 | S1-S2, S1-S3 | R1  |
| C4      | S2   | CE4-2  | LDP         | ge-0/0/7            | 412,423 | S1-S3, S2-S3 | R8  |
| C4      | S3   | CE4-3  | LDP         | ge-0/0/7            | 413.423 | S1-S3, S2-S3 | R6  |
| C5      | S1   | CE5-1  | BGP         | ge-0/0/7            | 512,513 | S1-S2, S1-S3 | R7  |
| C5      | S2   | CE5-2  | BGP         | ge-0/0/1            | 512,523 | S1-S2, S2-S3 | R3  |
| C5      | S3   | CE5-3  | BGP         | ge-0/0/8            | 513,523 | S1-S3, S2-S3 | R5  |

With this, we can follow to the configuration. 

Our goal is ensure the full connectivity between the sites, in other words, the customer wants a fully-mesh connection. 

This article will be very simple, and I will configure only one circuit of each customer to show you. 

Let's start on CE4-1 to CE4-2 circuit, using the VLAN 412. 

First, let's configure the interface: 
```
set interfaces ge-0/0/7 description to-CE4-1
set interfaces ge-0/0/7 flexible-vlan-tagging
set interfaces ge-0/0/7 encapsulation flexible-ethernet-services
set interfaces ge-0/0/7 unit 412 description S1-S2
set interfaces ge-0/0/7 unit 412 encapsulation vlan-ccc
set interfaces ge-0/0/7 unit 412 vlan-id 412
```
With this, we are tagging the VLAN in the PE-CE interface. 

Now, we need to transport this layer 2 segment across the network, trough the VPWS. In Junos, the VPWS Martini is referred as L2CKT. 
Let's configure this: 
```
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.412 virtual-circuit-id 412
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.412 description S1-S2
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.412 flow-label-transmit
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.412 flow-label-receive
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.412 mtu 1500
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.412 encapsulation-type ethernet-vlan
```
The mandatory parameters is: virtual-circuit-id, without vc-id we can't commit the configuration. In the LDP FEC 128 message, that contains the L2CKT data, we have encapsulation, MTU and VLAN information, these parameters must be matched to establish the VPWS. With any mismatch the circuit will be down and Junos will set the issue with some flags: 
```
Legend for connection status (St)
EI -- encapsulation invalid      NP -- interface h/w not present
MM -- mtu mismatch               Dn -- down
EM -- encapsulation mismatch     VC-Dn -- Virtual circuit Down
CM -- control-word mismatch      Up -- operational
VM -- vlan id mismatch           CF -- Call admission control failure
OL -- no outgoing label          IB -- TDM incompatible bitrate
NC -- intf encaps not CCC/TCC    TM -- TDM misconfiguration
BK -- Backup Connection          ST -- Standby Connection
CB -- rcvd cell-bundle size bad  SP -- Static Pseudowire
LD -- local site signaled down   RS -- remote site standby
RD -- remote site signaled down  HS -- Hot-standby Connection
XX -- unknown
```
Now, we can configure similary on the R8 and check the cicrcuit status: 
```
root@R1> show l2circuit connections interface ge-0/0/7.412
Layer-2 Circuit Connections:

Legend for connection status (St)
EI -- encapsulation invalid      NP -- interface h/w not present
MM -- mtu mismatch               Dn -- down
EM -- encapsulation mismatch     VC-Dn -- Virtual circuit Down
CM -- control-word mismatch      Up -- operational
VM -- vlan id mismatch           CF -- Call admission control failure
OL -- no outgoing label          IB -- TDM incompatible bitrate
NC -- intf encaps not CCC/TCC    TM -- TDM misconfiguration
BK -- Backup Connection          ST -- Standby Connection
CB -- rcvd cell-bundle size bad  SP -- Static Pseudowire
LD -- local site signaled down   RS -- remote site standby
RD -- remote site signaled down  HS -- Hot-standby Connection
XX -- unknown

Legend for interface status
Up -- operational
Dn -- down
Neighbor: 10.0.0.8
    Interface                 Type  St     Time last up          # Up trans
    ge-0/0/7.412(vc 412)      rmt   Up     Mar  6 15:14:36 2026           1
      Remote PE: 10.0.0.8, Negotiated control-word: Yes (Null)
      Incoming label: 29, Outgoing label: 20
      Negotiated PW status TLV: No
      Local interface: ge-0/0/7.412, Status: Up, Encapsulation: VLAN
        Description: S1-S2
      Flow Label Transmit: Yes, Flow Label Receive: Yes
```
And... it's ok!!!

To finish the delivery of this service, we can apply the same logic in the other PEs of the Customer 4. Let's go and ask the customer to check the connectivity. 

I can check on R1 both circuits, to Site 2 and Site 3. 
```
Layer-2 Circuit Connections:

Legend for connection status (St)
EI -- encapsulation invalid      NP -- interface h/w not present
MM -- mtu mismatch               Dn -- down
EM -- encapsulation mismatch     VC-Dn -- Virtual circuit Down
CM -- control-word mismatch      Up -- operational
VM -- vlan id mismatch           CF -- Call admission control failure
OL -- no outgoing label          IB -- TDM incompatible bitrate
NC -- intf encaps not CCC/TCC    TM -- TDM misconfiguration
BK -- Backup Connection          ST -- Standby Connection
CB -- rcvd cell-bundle size bad  SP -- Static Pseudowire
LD -- local site signaled down   RS -- remote site standby
RD -- remote site signaled down  HS -- Hot-standby Connection
XX -- unknown

Legend for interface status
Up -- operational
Dn -- down
Neighbor: 10.0.0.6
    Interface                 Type  St     Time last up          # Up trans
    ge-0/0/7.413(vc 413)      rmt   Up     Mar  6 15:12:01 2026           1
      Remote PE: 10.0.0.6, Negotiated control-word: Yes (Null)
      Incoming label: 28, Outgoing label: 20
      Negotiated PW status TLV: No
      Local interface: ge-0/0/7.413, Status: Up, Encapsulation: VLAN
        Description: S1-S3
      Flow Label Transmit: Yes, Flow Label Receive: Yes
Neighbor: 10.0.0.8
    Interface                 Type  St     Time last up          # Up trans
    ge-0/0/7.412(vc 412)      rmt   Up     Mar  6 15:14:36 2026           1
      Remote PE: 10.0.0.8, Negotiated control-word: Yes (Null)
      Incoming label: 29, Outgoing label: 20
      Negotiated PW status TLV: No
      Local interface: ge-0/0/7.412, Status: Up, Encapsulation: VLAN
        Description: S1-S2
      Flow Label Transmit: Yes, Flow Label Receive: Yes
```
Finally, let's see the customer connection test: 

Site 1 has connection with Site 2 and Site 3:
```
[admin@CE4-1] > ping 172.4.12.1
  SEQ HOST                                     SIZE TTL
    0 172.4.12.1                                 56  64
    1 172.4.12.1                                 56  64
    2 172.4.12.1                                 56  64
    sent=3 received=3 packet-loss=0% min-rtt=84us
   avg-rtt=95us max-rtt=111us

[admin@CE4-1] > ping 172.4.13.3
  SEQ HOST                                     SIZE TTL
    0 172.4.13.3                                 56  64
    1 172.4.13.3                                 56  64
    2 172.4.13.3                                 56  64
    sent=3 received=3 packet-loss=0% min-rtt=7ms16us
   avg-rtt=15ms729us max-rtt=21ms61us

```
And Site 2 has connection with Site 3 also! 
```
[admin@CE4-2] > ping 172.4.23.3
  SEQ HOST                                     SIZE TTL
    0 172.4.23.3                                 56  64
    1 172.4.23.3                                 56  64
    sent=2 received=2 packet-loss=0%
   min-rtt=11ms365us avg-rtt=30ms230us
   max-rtt=49ms95us
```
Ok!!! The service is delivered with success!!! 

The final configuration in the PEs is here: 
R1:
```
set interfaces ge-0/0/7 description to-CE4-1
set interfaces ge-0/0/7 flexible-vlan-tagging
set interfaces ge-0/0/7 encapsulation flexible-ethernet-services
set interfaces ge-0/0/7 unit 412 description S1-S2
set interfaces ge-0/0/7 unit 412 encapsulation vlan-ccc
set interfaces ge-0/0/7 unit 412 vlan-id 412
set interfaces ge-0/0/7 unit 413 description S1-S3
set interfaces ge-0/0/7 unit 413 encapsulation vlan-ccc
set interfaces ge-0/0/7 unit 413 vlan-id 413
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.412 virtual-circuit-id 412
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.412 description S1-S2
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.412 flow-label-transmit
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.412 flow-label-receive
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.412 mtu 1500
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.412 encapsulation-type ethernet-vlan
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.413 virtual-circuit-id 413
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.413 description S1-S3
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.413 flow-label-transmit
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.413 flow-label-receive
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.413 mtu 1500
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.413 encapsulation-type ethernet-vlan
```
R8:
```
set interfaces ge-0/0/7 description to-CE4-2
set interfaces ge-0/0/7 flexible-vlan-tagging
set interfaces ge-0/0/7 encapsulation flexible-ethernet-services
set interfaces ge-0/0/7 unit 412 description S1-S2
set interfaces ge-0/0/7 unit 412 encapsulation vlan-ccc
set interfaces ge-0/0/7 unit 412 vlan-id 412
set interfaces ge-0/0/7 unit 423 description S2-S3
set interfaces ge-0/0/7 unit 423 encapsulation vlan-ccc
set interfaces ge-0/0/7 unit 423 vlan-id 423
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/7.412 virtual-circuit-id 412
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/7.412 description S1-S2
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/7.412 flow-label-transmit
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/7.412 flow-label-receive
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/7.412 mtu 1500
set protocols l2circuit neighbor 10.0.0.1 interface ge-0/0/7.412 encapsulation-type ethernet-vlan
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.423 virtual-circuit-id 423
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.423 description S2-S3
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.423 flow-label-transmit
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.423 flow-label-receive
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.423 mtu 1500
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.423 encapsulation-type ethernet-vlan
```
R6:
```
set interfaces ge-0/0/7 description to-CE4-3
set interfaces ge-0/0/7 flexible-vlan-tagging
set interfaces ge-0/0/7 encapsulation flexible-ethernet-services
set interfaces ge-0/0/7 unit 413 description S1-S3
set interfaces ge-0/0/7 unit 413 encapsulation vlan-ccc
set interfaces ge-0/0/7 unit 413 vlan-id 413
set interfaces ge-0/0/7 unit 423 description S2-S3
set interfaces ge-0/0/7 unit 423 encapsulation vlan-ccc
set interfaces ge-0/0/7 unit 423 vlan-id 423
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.423 virtual-circuit-id 423
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.423 description S2-S3
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.423 flow-label-transmit
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.423 flow-label-receive
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.423 mtu 1500
set protocols l2circuit neighbor 10.0.0.8 interface ge-0/0/7.423 encapsulation-type ethernet-vlan
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.413 virtual-circuit-id 413
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.413 description S1-S3
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.413 flow-label-transmit
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.413 flow-label-receive
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.413 mtu 1500
set protocols l2circuit neighbor 10.0.0.6 interface ge-0/0/7.413 encapsulation-type ethernet-vlan
```

Ok, you see that Martini VPWS was very easy, right? I think Martini is more simple and objective to configure, but with BGP we can spare some LDP connections, and turn all the things more scalable. The label information of VPWS Kompella is changed by BGP, and in the VPWS Martini the label information is changed by LDP FEC 128 message, that need a LDP peering between the PEs, fortunately Junos try to establish the T-LDP (Target LDP) peering when you configure a VPWS or VPLS Martini, in other vendors could be necessary a manual T-LDP configuration, that is more painful. 

Everything is cleared here? I think so. 

Let's go to the final part of this delivery. 

First things first. We need to enable the l2vpn signalling in our BGP mesh. 
RR:
```
set protocols bgp group iBGP-AS65020-East family l2vpn signaling nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-East family l2vpn signaling no-install
set protocols bgp group iBGP-AS65020-West family l2vpn signaling nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-West family l2vpn signaling no-install
```
PEs:
```
set protocols bgp group iBGP-AS65020-East family l2vpn signaling
or
set protocols bgp group iBGP-AS65020-West family l2vpn signaling
```

Now, with full connectivity, let's configure the VPWS on PEs, this time I'll apply the configuration on R7, but here I'll configure the two remote site in one time! 
```
set interfaces ge-0/0/7 description to-CE5-1
set interfaces ge-0/0/7 flexible-vlan-tagging
set interfaces ge-0/0/7 encapsulation flexible-ethernet-services
set interfaces ge-0/0/7 unit 512 description S1-S2
set interfaces ge-0/0/7 unit 512 encapsulation vlan-ccc
set interfaces ge-0/0/7 unit 512 vlan-id 512
set interfaces ge-0/0/7 unit 513 description S1-S3
set interfaces ge-0/0/7 unit 513 encapsulation vlan-ccc
set interfaces ge-0/0/7 unit 513 vlan-id 513

set routing-instances L2VPN-CE5-S1 instance-type l2vpn
set routing-instances L2VPN-CE5-S1 protocols l2vpn site s1 interface ge-0/0/7.512 remote-site-id 2
set routing-instances L2VPN-CE5-S1 protocols l2vpn site s1 interface ge-0/0/7.513 remote-site-id 3
set routing-instances L2VPN-CE5-S1 protocols l2vpn site s1 site-identifier 1
set routing-instances L2VPN-CE5-S1 protocols l2vpn site s1 mtu 1500
set routing-instances L2VPN-CE5-S1 protocols l2vpn encapsulation-type ethernet-vlan
set routing-instances L2VPN-CE5-S1 description L2VPN-CE5-S1
set routing-instances L2VPN-CE5-S1 interface ge-0/0/7.512
set routing-instances L2VPN-CE5-S1 interface ge-0/0/7.513
set routing-instances L2VPN-CE5-S1 route-distinguisher 10.0.0.7:5
set routing-instances L2VPN-CE5-S1 vrf-target target:65020:5
```
Pay attention on the configuration here, I have a local-site identifier, and I can specify multiple remote-site identifiers in the site configuration. You got it? We can configure the full mesh connectivity in a easier way than VPWS Martini!!! The unique problem, is documenting the site-id, but I trust you, you can do it. 

Ok, with this defined, our router will advertise the bgp.l2vpn route to RR and if our remote PE have the correct RT configured, it will receive the route and the opposite will happens also. 
Let's check this:
```
root@R7> show route table L2VPN-CE5-S1.

L2VPN-CE5-S1.l2vpn.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both
...................
 10.0.0.3:5:2:1/96 (1 entry, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 10.0.0.3:5
                Next hop type: Indirect, Next hop index: 0
                Address: 0x80b1d94
                Next-hop reference count: 12
                Kernel Table Id: 0
                Source: 10.0.0.0
                Protocol next hop: 10.0.0.3
                Indirect next hop: 0x2 no-forward INH Session ID: 0
                Indirect next hop: INH non-key opaque: 0x0 INH key opaque: 0x0
                State: <Secondary Active Int Ext>
                Local AS: 65020 Peer AS: 65020
                Age: 21:11      Metric2: 1
                Validation State: unverified
                Task: BGP_65020.10.0.0.0
                Announcement bits (1): 0-L2VPN-CE5-S1-l2vpn
                AS path: I  (Originator)
                Cluster list:  0.0.0.2
                Originator ID: 10.0.0.3
                Communities: target:65020:5 Layer2-info: encaps: VLAN, control flags:[0x2] Control-Word, mtu: 1500, site preference: 100
                Import Accepted
                Label-base: 25, range: 4, status-vector: 0x0, offset: 1
                Localpref: 100
                Router ID: 10.0.0.0
                Primary Routing Table: bgp.l2vpn.0
                Thread: junos-main
...................
10.0.0.5:5:3:1/96
                   *[BGP/170] 00:08:46, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.20 via ge-0/0/3.0, Push 72
                       to 10.200.0.23 via ge-0/0/4.0, Push 89
10.0.0.7:5:1:1/96
                   *[L2VPN/170/-101] 1w4d 04:47:41, metric2 1
                       Indirect
10.0.0.7:5:1:9/96
                   *[L2VPN/170/-101] 1w4d 04:47:41, metric2 1
                       Indirect

L2VPN-CE5-S1.l2id.0: 4 destinations, 8 routes (4 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1/32
                   *[L2VPN/170/-101] 1w4d 04:47:41, metric2 1
                       Indirect
                    [L2VPN/170/-101] 1w4d 04:47:41, metric2 1
                       Indirect
                    [L2VPN/175] 00:08:46
                    >  via ge-0/0/7.512, Pop       Offset: 4
                    [L2VPN/175] 00:08:46
                    >  via ge-0/0/7.500, Pop       Offset: 4
                    [L2VPN/175] 00:08:46
                    >  via ge-0/0/7.513, Pop       Offset: 4
2/32
                   *[BGP/170] 00:08:46, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.23 via ge-0/0/4.0, label-switched-path R7-R2-A
                       to 10.200.0.20 via ge-0/0/3.0, label-switched-path Bypass->10.200.0.23
3/32
                   *[BGP/170] 00:08:46, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.20 via ge-0/0/3.0, Push 72
                       to 10.200.0.23 via ge-0/0/4.0, Push 89

```
In the routing tables, we can see all L2VPN routes. The structure of the route is basically ```router-id:site-id:label-block/96```. When a router receive the route of the remote PE, it knows how to forward the traffic, and what label use to forward this VPN traffic. Sure, here all the things is simplificated to the operator see that, but, basically, when the route is received with a RT of some routing-instance, these routes is injected into the ```L2VPN.l2id.0``` table. And the sites are identified accordingly. 

With all the routes received, we can check the circuit status: 
```
root@R7> show l2vpn connections
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

Instance: L2VPN-CE5-S1
Edge protection: Not-Primary
  Local site: s1 (1)
    connection-site           Type  St     Time last up          # Up trans
    2                         rmt   Up     Apr  6 16:28:40 2026           1
      Remote PE: 10.0.0.3, Negotiated control-word: Yes (Null)
      Incoming label: 20, Outgoing label: 25
      Local interface: ge-0/0/7.512, Status: Up, Encapsulation: VLAN
      Flow Label Transmit: No, Flow Label Receive: No
    3                         rmt   Up     Apr  6 16:28:40 2026           1
      Remote PE: 10.0.0.5, Negotiated control-word: Yes (Null)
      Incoming label: 21, Outgoing label: 17
      Local interface: ge-0/0/7.513, Status: Up, Encapsulation: VLAN
      Flow Label Transmit: No, Flow Label Receive: No
```
And... Voilà!!! All the sites are connected, with the L2VPN routes received I know that all sites are configured correctly. 

Let's ask the customer to check the connectivity between the sites: 
Site 1 with Site 2 and 3:
```
[admin@CE5-1] > ping 172.5.12.2
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.5.12.2                                 56  64 40ms124us
    1 172.5.12.2                                 56  64 24ms248us
    2 172.5.12.2                                 56  64 33ms981us
    sent=3 received=3 packet-loss=0% min-rtt=24ms248us avg-rtt=32ms784us
   max-rtt=40ms124us

[admin@CE5-1] > ping 172.5.13.3
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.5.13.3                                 56  64 35ms860us
    1 172.5.13.3                                 56  64 15ms577us
    2 172.5.13.3                                 56  64 6ms949us
    sent=3 received=3 packet-loss=0% min-rtt=6ms949us avg-rtt=19ms462us
   max-rtt=35ms860us

```
Site 2 with Site 3:
```
[admin@CE5-2] > ping 172.5.23.3
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 172.5.23.3                                 56  64 42ms375us
    1 172.5.23.3                                 56  64 11ms382us
    2 172.5.23.3                                 56  64 7ms671us
    sent=3 received=3 packet-loss=0% min-rtt=7ms671us avg-rtt=20ms476us
   max-rtt=42ms375us
```
OK! 

Our goal is accomplished more on time. 

In the next article I'll made the VPLS configuration. If you want to imagine some jokes to do, you can use L2CKTs and VPLS together, since they both use LDP FEC 128 messages! I won't say more about this, I only want to plant the idea in your mind, hahahah. 

See you soon! 
