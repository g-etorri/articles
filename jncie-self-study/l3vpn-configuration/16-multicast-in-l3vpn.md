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
root@RR> show route table bgp.mvpn.0           

bgp.mvpn.0: 5 destinations, 5 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

1:10.0.0.1:200:10.0.0.1/240               
                   *[BGP/170] 00:49:10, localpref 100
                      AS path: I, validation-state: unverified
                    >  to 10.0.0.1
1:10.0.0.2:200:10.0.0.2/240               
                   *[BGP/170] 00:45:02, localpref 100
                      AS path: I, validation-state: unverified
                    >  to 10.0.0.2
1:10.0.0.4:201:10.0.0.4/240               
                   *[BGP/170] 00:41:05, localpref 100
                      AS path: I, validation-state: unverified
                    >  to 10.0.0.4
1:10.0.0.5:201:10.0.0.5/240               
                   *[BGP/170] 00:40:45, localpref 100
                      AS path: I, validation-state: unverified
                    >  to 10.0.0.5
1:10.0.0.7:201:10.0.0.7/240               
                   *[BGP/170] 00:40:32, localpref 100
                      AS path: I, validation-state: unverified
                    >  to 10.0.0.7      
```
This route is used for autodiscovery of the MVPN. Do you remember the MVPN routes? 

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

This table will facilitate our undestanding about MVPN. 
