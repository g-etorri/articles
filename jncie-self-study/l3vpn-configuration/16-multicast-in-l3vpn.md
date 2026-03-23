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

Do you remember the topology of Customer 2? We have a hub-and-spoke topology, and R1 and R2 are the HUB PEs. In this case, the HUB PEs will act as RP and will be the sender sites. The SPOKE PEs will be the receiver sites. To forward this traffic to the SPOKE PEs, we'll use MLSPs! During the configuration, I'll define some constraints to learn something more. 

Starting, let's configure the MVPN on the HUB routing-instance: 
* On R1 and R2 we'll use a new loopback address (10.2.0.254), acting as anycast RP. 
* We need to configure the PIM on PE-CE interface to have communication with the customer, and define the RP address of the group range 239.0.0.0/24 used by the customer. 
* The MVPN define the MVPN, sure. Here we are defining that our PE will be the sender site and by default, , and we defined different RTs for the MVPN.
* We created a P2MP template to use as LSPs of our MVPN.
* Finally, we need to add the chassis configuration to de-encapsulate the PIM messages.
```
set interfaces lo0 unit 1 family inet address 10.2.1.254/32 primary
set interfaces lo0 unit 1 family inet address 10.2.0.254/32

set routing-instances VRF-C2-HUB protocols pim rp local address 10.2.0.254
set routing-instances VRF-C2-HUB protocols pim rp local group-ranges 239.0.0.0/24
set routing-instances VRF-C2-HUB protocols pim interface all

set routing-instances VRF-C2-HUB protocols mvpn sender-site
set routing-instances VRF-C2-HUB protocols mvpn route-target import-target target target:65020:204
set routing-instances VRF-C2-HUB protocols mvpn route-target export-target target target:65020:204

set protocols mpls label-switched-path lsp-mcast-p2mp-template template
set protocols mpls label-switched-path lsp-mcast-p2mp-template p2mp

set routing-instances VRF-C2-HUB provider-tunnel rsvp-te label-switched-path-template lsp-mcast-p2mp-template

set chassis fpc 0 pic 0 tunnel-services
```
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

VRF-C2-HUB.mvpn.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.0.0.1:200:10.0.0.1/240               
                   *[MVPN/70] 01:02:31, metric2 1
                       Indirect
1:10.0.0.2:200:10.0.0.2/240               
                   *[BGP/170] 00:58:21, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 0
1:10.0.0.4:201:10.0.0.4/240               
                   *[BGP/170] 00:54:24, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 0
1:10.0.0.5:201:10.0.0.5/240               
                   *[BGP/170] 00:54:04, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 10106, Push 10107, Push 10108(top)
                       to 10.200.0.5 via ge-0/0/3.0, Push 10106, Push 10107, Push 10108(top)
1:10.0.0.7:201:10.0.0.7/240               
                   *[BGP/170] 00:53:51, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 45     
```
Note: The route format of MVPN route type 1 is ROUTE-TYPE:ROUTE-DISTINGUISHER:ORIGIN.
This route is used for autodiscovery of the MVPN. Do you remember the MVPN routes? I don't. 

I'll give you a table with the routes:

| Route Type | Name | Who Sends? | Description |
| - | - | - | - |
| Type 1 | Intra-AS I-PMSI | All PEs | Used for MVPN autodiscovery |
| Type 2 | Inter-AS I-PMSI | ASBRs | Used in Inter-AS VPNs scenario, Option B or C |
| Type 3 | S-PMSI AD | Sender PE | Used to create a selective tunnel |
| Type 4 | Leaf AD | Receiver PE | Used to respond the route type 3, to build a selective tree |
| Type 5 | Source Active | Sender PE | Used to inform the PEs that the source started the stream |
| Type 6 | Shared Tree Join | Receiver PE | Used to join a shared tree, in other words, to receive the stream from RP |
| Type 7 | Source Tree Join | Receiver PE | Used to join a source based tree, the PE wants to receive the stream from the source |

This table will facilitate our understanding about MVPN. 

Now, checking our MVPN instance, we can see the members:
```
root@R1> show mvpn instance all 

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
root@R1> show pim neighbors instance all 
B = Bidirectional Capable, G = Generation Identifier
H = Hello Option Holdtime, L = Hello Option LAN Prune Delay,
P = Hello Option DR Priority, T = Tracking Bit,
A = Hello Option Join Attribute

Instance: PIM.VRF-C2-HUB
Interface           IP V Mode        Option       Uptime Neighbor addr
ge-0/0/8.200         4 2             HPLGT       01:09:47 10.2.1.2       
```

Ok, now we are ready to forward the multicast traffic. Let's create a stream on our network and some members of the group. 
```

```
