# SR-MPLS Configuration

Hello guys! After mastering RSVP protection, it's time to dive into SR-MPLS. SR is a game-changer because it simplifies the control plane by removing the need for LDP/RSVP, using the IGP (ISIS or OSPF) to distribute labels.

The topology you already know:
<img width="1033" height="783" alt="image" src="https://github.com/user-attachments/assets/7ab718e7-c78f-420c-8b23-978f8b04e62d" />

Now, follow the same style of the previous post, let's list our tasks:
* Configure SR-MPLS in all routers, defining a block of 5000 labels and ensuring that the final label was 15000. Using 100+ Router number as Segment-ID of the router.
* In R5, make a static SR-TE to R1 that passes trough R8. In R1, made a SR-TE to R5 that passes trough R7. Don't specific the SIDs explicitly in the path.
* In R2, made a static SR-TE to R5 that passes trough R4. Without specificating the SIDs explicitly in the path. In R5, made a static SR-TE to R2, passing trough R1 (ensure that the R1 allocate the labels statically between 40000 and 45000) and arriving in R2 trough the interface ge-0/0/1 of the LAG. 
* In all routers of the network, enable TI-LFA link and node-protection for the routes in the inet.0 and inet.3.
* Configure LDP tunneling in the SR-TEs between R1 and R5, and R2 and R5.
* In R3, confiture a dynamic SR-TE to R4 that passes trough R2. Ensure that this dynamic path only be used if don't have LDP routes to R4.
* In R3, configure the microloop avoidance, and ensure that the convergence paths will be removed after 10 seconds.

Let's start our journey enabling the segment routing and definining the label block and the segment-id of each router. 
To SR work, we need to define the enhanced-ip in the chassis of the routers, and create the configuration of source-packet-routing. 
```
set chassis network-services enhanced-ip
set protocols source-packet-routing
```
Now, inside the IGP configuration we can define the label block and the segmeent-id of the router, here I'm configuring in the R1, you must replicate the configuration similarly in the other routers. Note that segment-id is 100 + Router number, in other words, R1 is 101, R2 is 102 and so on. 
```
set protocols isis source-packet-routing srgb start-label 10001
set protocols isis source-packet-routing srgb index-range 5000
set protocols isis source-packet-routing node-segment ipv4-index 101
```
Here we have a prank, naturally you think if we have to finish the label block in 15000, we will start this in 10000, but this is wrong, the 0 is still a label. So, we need to start in 10001 to achieve this. 

Ok, let's check the results:
```
root@R1> show mpls label usage
Label space Total   Available        Applications
LSI         69609   69609  (100.00%) BGP/LDP VPLS with no-tunnel-services, BGP L3VPN with vrf-table-label
Block       199936  199936 (100.00%) BGP/LDP VPLS with tunnel-services, BGP L2VPN
Dynamic     487936  487905 (99.99% ) RSVP, LDP, PW, L3VPN, RSVP-P2MP, LDP-P2MP, MVPN, EVPN, BGP, SPRING-TE
Static      48576   48576  (100.00%) Static LSP, Static PW
```
We can see that we don't have any label block yet, we need to restart de rpd to the changes take effect. 
```
root@R1> restart routing
Routing protocols process signalled but still running, waiting 28 seconds more
Routing protocols process started, pid 48721

root@R1> show isis overview | match alloc
    SRGB Block Allocation: Success

root@R1> show mpls label usage
Label space Total   Available        Applications
LSI         994984  994976 (100.00%) BGP/LDP VPLS with no-tunnel-services, BGP L3VPN with vrf-table-label
Block       994984  994976 (100.00%) BGP/LDP VPLS with tunnel-services, BGP L2VPN
Dynamic     994984  994976 (100.00%) RSVP, LDP, PW, L3VPN, RSVP-P2MP, LDP-P2MP, MVPN, EVPN, BGP, SPRING-TE
Static      48576   48576  (100.00%) Static LSP, Static PW
Effective Ranges
Range name  Shared with Start   End
Dynamic     16      10000
Dynamic     15001   999999
Static      1000000 1048575
SRGB        10001   15000    ISIS
Configured Ranges
Range name  Shared with Start   End
Dynamic     16      10000
Dynamic     15001   999999
Static      1000000 1048575
SRGB        10001   15000    ISIS
```
Now, our label block is defined, we need to do this in all routers of our network, except the RR, sure. 

Let's verify the ISIS database, now our IGP distribute the label information! 
```
root@R1> show isis database detail | match Index 
  IPV4 Index: 101
    Start Index : 0, Size : 5000, Label-Range: [ 10001, 15000 ]
  IPV4 Index: 102
    Start Index : 0, Size : 5000, Label-Range: [ 10001, 15000 ]
  IPV4 Index: 103
    Start Index : 0, Size : 5000, Label-Range: [ 10001, 15000 ]
  IPV4 Index: 104
    Start Index : 0, Size : 5000, Label-Range: [ 10001, 15000 ]
  IPV4 Index: 105
    Start Index : 0, Size : 5000, Label-Range: [ 10001, 15000 ]
  IPV4 Index: 106
    Start Index : 0, Size : 5000, Label-Range: [ 10001, 15000 ]
  IPV4 Index: 107
    Start Index : 0, Size : 5000, Label-Range: [ 10001, 15000 ]
  IPV4 Index: 108
    Start Index : 0, Size : 5000, Label-Range: [ 10001, 15000 ]
```
Everything is ok, label blocks defined and the segment-id is correct! 

Now, let's make a SR-TE, I particularly don't like make this manually, if you will adopt the SR-MPLS it's time to think in a controller to made the TE decisions. Or, continue with RSVP, this is a particular opinion btw. 

To accomplish the goal without specify the SIDs, we need to enable the auto-translate in the segment-list. This knob translate the IP address in SID automatically to define the segment-list. Then, we need to set the segment-list into the SR-TE. 
```
set protocols source-packet-routing segment-list sl-r1 auto-translate
set protocols source-packet-routing segment-list sl-r1 1 ip-address 10.0.0.8
set protocols source-packet-routing segment-list sl-r1 1 label-type node
set protocols source-packet-routing segment-list sl-r1 2 ip-address 10.0.0.1
set protocols source-packet-routing segment-list sl-r1 2 label-type node

set protocols source-packet-routing source-routing-path to-R1 to 10.0.0.1
set protocols source-packet-routing source-routing-path to-R1 primary sl-r1
```
And voilà, this is like a LSP with an explicit-path. This configuration was made in R5, we need to replicate similarly in R1. 

Let's check the outputs to confirm if our SR-TE is computed correctly. 
```
root@R5> show spring-traffic-engineering lsp detail 
E = Entropy-label Capability

Name: to-R1
  Tunnel-source: Static configuration
  Tunnel Forward Type: SRMPLS
  To: 10.0.0.1
  Te-group-id: 0
  State: Up
    Path: sl-r1
    Path Status: NA
    Outgoing interface: NA
    Auto-translate status: Enabled Auto-translate result: Success
    Compute Status:Disabled , Compute Result:N/A , Compute-Profile Name:N/A
    BFD status: N/A BFD name: N/A
    BFD remote-discriminator: N/A
    Segment ID : 128 
    ERO Valid: true
      SR-ERO hop count: 2
        Hop 1 (Loose): 
          NAI: IPv4 Node ID, Node address: 10.0.0.8
          SID type: 20-bit label, Value: 10109
        Hop 2 (Loose): 
          NAI: IPv4 Node ID, Node address: 10.0.0.1
          SID type: 20-bit label, Value: 10102
```
You can marvel the SID, because the R8 is 10109 and for the eyes is most beautiful if would 10108, but this is the operation of SR, here we have a sum of Node-SID+Label Block, with the label block starting in 10001, we have 10001 + 108 = 10109, you got it? 

In the FIB/RIB looks like this:
```
root@R5> show route table inet.3 protocol spring-te 

inet.3: 20 destinations, 36 routes (8 active, 0 holddown, 16 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.1/32        *[SPRING-TE/8] 03:19:24, metric 1, metric2 16777224
                    >  to 10.200.0.19 via ge-0/0/2.0, Push 10102
                       to 10.200.0.17 via ae0.0, Push 10102, Push 10109, Push 10108(top)
```
Our SR-TE was computed with success! 

Now let's made another communication with SR-TE, between R2 and R5. In R2 we'll configure in the same way that we configured now, with the segment-list translating IP addresses. But, this time in R5 we'll made a bit different. Let's specify the SIDs of the path, and let's forward the traffic to R2 trough a specific interface of the LAG between R1 and R2. 

First, we need to create a static label entry in R1, this way, R1 can forward the traffic only trough ge-0/0/0 in the LAG. The tasks ask us to ensure that R1 will use a static label range 40000-45000. 

So, we create the static-label-range, then, create a static label entry, specifying the next-hop address, the output interface and the label action. R1 as the penultimate hop, and will do the penultimate-hop-popping.
```
set protocols mpls label-range static-label-range 40000 45000

set protocols mpls static-label-switched-path via-ge-0/0/0 transit 40002 next-hop 10.200.0.1
set protocols mpls static-label-switched-path via-ge-0/0/0 transit 40002 member-interface ge-0/0/0
set protocols mpls static-label-switched-path via-ge-0/0/0 transit 40002 pop
```
With this defined, we can configure the SR-TE in R5 specifying the stacks` labels, first the R1 label, then the interface specific label:
```
set protocols source-packet-routing segment-list sl-r2 1 label 10102
set protocols source-packet-routing segment-list sl-r2 2 label 40002
set protocols source-packet-routing source-routing-path to-R2 ldp-tunneling
set protocols source-packet-routing source-routing-path to-R2 to 10.0.0.2
set protocols source-packet-routing source-routing-path to-R2 primary sl-r2
```
Now, let's check our SR-TE:
```

Name: to-R2
  Tunnel-source: Static configuration
  Tunnel Forward Type: SRMPLS
  To: 10.0.0.2
  Te-group-id: 0
  State: Up
    Path: sl-r2
    Path Status: NA
    Outgoing interface: NA
    Auto-translate status: Disabled Auto-translate result: N/A
    Compute Status:Disabled , Compute Result:N/A , Compute-Profile Name:N/A
    BFD status: N/A BFD name: N/A
    BFD remote-discriminator: N/A
    Segment ID : 128 
    ERO Valid: true
      SR-ERO hop count: 2
        Hop 1 (Strict): 
          NAI: None
          SID type: 20-bit label, Value: 10102
        Hop 2 (Strict): 
          NAI: None                     
          SID type: 20-bit label, Value: 40002
```
You can see that we don't have auto-translate this time, we are computating the SR-TE without translate IP address in labels. 

In the FIB/RIB looks like this:
```
root@R5> show route table inet.3 protocol spring-te 

inet.3: 20 destinations, 36 routes (8 active, 0 holddown, 16 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32         [SPRING-TE/8] 00:04:02, metric 1, metric2 16777229
                       to 10.200.0.14 via ge-0/0/3.0, Push 40002, Push 10102(top)
                    >  to 10.200.0.19 via ge-0/0/2.0, Push 40002, Push 10102(top)
```
We have the label stack as we defined. 

Now, let's improve our convergence during fails in the network using TI-LFA and node-protection.
The configuration is very simple, we need to enable the TI-LFA on the interfaces, then for inet.3 routes we must include the knob use-post-convergence-lfa, and to use this backup routes also in inet.0, we need to include use-source-packet-routing. 

We can apply this configuration in all routers of the network: 
```
set protocols isis interface all level 1 post-convergence-lfa
set protocols isis backup-spf-options use-post-convergence-lfa
set protocols isis backup-spf-options use-source-packet-routing
```
To verify if this is working properly, let's see the outputs: 
```
root@R1> show isis overview | match Post
  Post Convergence Backup: Enabled

root@R1> show isis interface ae0.0 extensive
IS-IS interface database:
ae0.0
  Index: 342, State: 0x6, Circuit id: 0x1, Circuit type: 1
  LSP interval: 100 ms, CSNP interval: 20 s, Loose Hello padding, IIH max size: 1492
  Adjacency advertisement: Advertise, Layer2-map: Disabled
  Interface Group Holddown Delay: 20 s, remaining: 0 s
  Level 1
    Adjacencies: 1, Priority: 64, Metric: 5
    Hello Interval: 9.000 s, Hold Time: 27 s
    Post convergence Protection:Enabled, Fate sharing: No, Srlg: No, Node cost: 0
    IPV6 UnicastMetric: 5
```
We can see in the interfaces the Post Convergence Protection enabled. 

In the RIB we can see the backup path calculated with the TI-LFA:
```
root@R1> show route table inet.3 protocol isis 

inet.3: 15 destinations, 28 routes (7 active, 0 holddown, 11 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32         [L-ISIS/14] 03:29:38, metric 5
                    >  to 10.200.0.1 via ae0.0
                       to 10.200.0.3 via ge-0/0/2.0, Push 10103, Push 10104(top)
```
Ok, everything looks good so far. 

We have reach in the half of the tasks, let's make a checkpoint. You can pick up a coffee to follow for the another half, let's go man, you want it. 

* ~~Configure SR-MPLS in all routers, defining a block of 5000 labels and ensuring that the final label was 15000. Using 100+ Router number as Segment-ID of the router.~~
* ~~In R5, make a static SR-TE to R1 that passes trough R8. In R1, made a SR-TE to R5 that passes trough R7. Don't specific the SIDs explicitly in the path.~~
* ~~In R2, made a static SR-TE to R5 that passes trough R4. Without specificating the SIDs explicitly in the path. In R5, made a static SR-TE to R2, passing trough R1 (ensure that the R1 allocate the labels statically between 40000 and 45000) and arriving in R2 trough the interface ge-0/0/1 of the LAG.~~
* ~~In all routers of the network, enable TI-LFA link and node-protection for the routes in the inet.0 and inet.3.~~
* Configure LDP tunneling in the SR-TEs between R1 and R5, and R2 and R5.
* In R3, confiture a dynamic SR-TE to R4 that passes trough R2. Ensure that this dynamic path only be used if don't have LDP routes to R4.
* In R3, configure the microloop avoidance, and ensure that the convergence paths will be removed after 10 seconds.

Now, let's make another connection between LDP islands, now, using SR-TE!!!
Let's take advantage of the SR-TE created previous, and add the knob ldp-tunneling, yes, easy peasy. I made this on R1 and replicate similarly on the rest:
```
set protocols source-packet-routing source-routing-path to-R5 ldp-tunneling
```
Let me check if the LDP tunneling is working now: 
```
Name: to-R5
  Tunnel-source: Static configuration
  Tunnel Forward Type: SRMPLS
  To: 10.0.0.5
  Te-group-id: 0
  State: Up
  LDP-tunneling enabled
    Path: sl-r5
    Path Status: NA
    Outgoing interface: NA
    Auto-translate status: Enabled Auto-translate result: Success
    Compute Status:Disabled , Compute Result:N/A , Compute-Profile Name:N/A
    BFD status: N/A BFD name: N/A
    BFD remote-discriminator: N/A
    Segment ID : 128 
    ERO Valid: true
      SR-ERO hop count: 3
        Hop 1 (Loose): 
          NAI: IPv4 Node ID, Node address: 10.0.0.7
          SID type: 20-bit label, Value: 10108
        Hop 2 (Loose): 
          NAI: IPv4 Node ID, Node address: 10.0.0.6
          SID type: 20-bit label, Value: 10107
        Hop 3 (Loose): 
          NAI: IPv4 Node ID, Node address: 10.0.0.5
          SID type: 20-bit label, Value: 10106

root@R1> show ldp session 10.0.0.5 extensive 
Address: 10.0.0.5, State: Operational, Connection: Open, Hold time: 23
  Session ID: 10.0.0.1:0--10.0.0.5:0
  Next keepalive in 1 seconds
  Passive, Maximum PDU: 4096, Hold time: 30, Neighbor count: 1
  Neighbor types: auto-targeted auto-tunneled
  Keepalive interval: 10, Connect retry interval: 1
  Local address: 10.0.0.1, Remote address: 10.0.0.5
```
And, it's ok!!! In the SPRING LSP we can see the LDP-tunneling enabled line. And take a special look in the "Neighbor types: auto-targeted auto-tunneled", this line tell us if the LDP tunneling is working properly. I think I forgot this output in the RSVP article, but ok, we saw this in other manner. 

Now, we need to configure a SR-TE dynamic. But, what's the difference between a dynamic SR-TE and a static SR-TE?

The static SR-TE is basically tell to the router what the label stack that it needs to push. And the dynamic SR-TE, we tell to the router to compute a path with stricts or loose hops, consulting the CSPF. Dynamically the router will push de SR labels to compute a path passing trough the explicit hops, or using a specific admin-group. 

In this example, we only need to compute a path passing trough R2 strictly, then arrive in R4. And, to this SPRING-TE will be used only in LDP fails, we can change the preference of SPRING-TE. 
```
set protocols source-packet-routing segment-list to-R4-via-R2 compute
set protocols source-packet-routing segment-list to-R4-via-R2 1 ip-address 10.0.0.2
set protocols source-packet-routing segment-list to-R4-via-R2 1 strict
set protocols source-packet-routing segment-list to-R4-via-R2 2 ip-address 10.0.0.4
set protocols source-packet-routing segment-list to-R4-via-R2 2 loose

set protocols source-packet-routing compute-profile to-r4-computed compute-segment-list to-R4-via-R2

set protocols source-packet-routing source-routing-path to-R4 to 10.0.0.4
set protocols source-packet-routing source-routing-path to-R4 primary to-r4-computed compute computed-path

set protocols source-packet-routing preference 10
```
Now, let's look the outputs:
```
Name: to-R4
  Tunnel-source: Static configuration
  Tunnel Forward Type: SRMPLS
  To: 10.0.0.4
  Te-group-id: 0
  State: Up
    Path: to-r4-computed
    Path Status: NA
    Outgoing interface: NA
    Auto-translate status: Disabled Auto-translate result: N/A
    Compute Status:Enabled , Compute Result:success , Compute-Profile Name:to-r4-computed
    Total number of computed paths: 1
    Segment ID : 128 
    Computed-path-index: 1
      BFD status: N/A BFD name: N/A
      BFD remote-discriminator: N/A
      TE metric: 25, IGP metric: 25
      Delay metrics: Min: 50331645, Max: 50331645, Avg: 50331645
      Metric optimized by type: TE
      computed segments count: 2
        computed segment : 1 (computed-node-segment): 
          node segment label: 10102
          router-id: 10.0.0.1 ::1
        computed segment : 2 (computed-node-segment): 
          node segment label: 10105
          router-id: 10.0.0.4 ::1

root@R3> show route 10.0.0.4 table inet.3 

inet.3: 73 destinations, 105 routes (73 active, 0 holddown, 14 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.4/32        *[LDP/9] 03:54:14, metric 10
                    >  to 10.200.0.11 via ge-0/0/4.0, Push 0
                       to 10.200.0.6 via ge-0/0/3.0, Push 34
                    [SPRING-TE/10] 03:54:13, metric 1, metric2 10
                    >  to 10.200.0.6 via ge-0/0/3.0, Push 10105, Push 10102(top)
                       to 10.200.0.11 via ge-0/0/4.0, Push 10105, Push 10102(top)
```
The path computed is R3 -> R2 -> R1 -> R4. And in the RIB the SPRING-TE have a bigger prefence than LDP. 

Bonus: Let's include a admin-group constraint now. 

