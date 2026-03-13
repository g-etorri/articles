# SR-MPLS Configuration

Hello guys! After mastering RSVP protection, it's time to dive into SR-MPLS. SR is a game-changer because it simplifies the control plane by removing the need for LDP/RSVP, using the IGP (IS-IS or OSPF) to distribute labels.

The topology you already know:
<img width="1033" height="783" alt="image" src="https://github.com/user-attachments/assets/7ab718e7-c78f-420c-8b23-978f8b04e62d" />

Following the same style as the previous post, let's list our tasks:
* Configure SR-MPLS on all routers, defining a block of 5,000 labels and ensuring that the final label is 15,000. Use 100 + Router Number as the Segment-ID of the router.
* On R5, create a static SR-TE to R1 that passes through R8. On R1, create a SR-TE to R5 that passes through R7. Do not specify the SIDs explicitly in the path.
* On R2, create a static SR-TE to R5 that passes through R4 without specifying the SIDs explicitly in the path. On R5, create a static SR-TE to R2, passing through R1 (ensure that R1 allocates the labels statically between 40,000 and 45,000) and arriving at R2 through interface ge-0/0/1 of the LAG.
* On all routers in the network, enable TI-LFA link and node protection for routes in inet.0 and inet.3.
* Configure LDP tunneling in the SR-TEs between R1 and R5, and R2 and R5.
* On R3, configure a dynamic SR-TE to R4 that passes through R2. Ensure that this dynamic path is only used if there are no LDP routes to R4.
* On R3, configure microloop avoidance and ensure that the convergence paths are removed after 10 seconds.

Let's start our journey by enabling segment routing and defining the label block and the segment-id of each router.
For SR to work, we need to define enhanced-ip in the chassis of the routers and create the source-packet-routing configuration.
```
set chassis network-services enhanced-ip
set protocols source-packet-routing
```
Now, inside the IGP configuration, we can define the label block and the segment-id of the router. Here, I'm configuring R1; you must replicate the configuration similarly on the other routers. Note that the segment-id is 100 + Router number—in other words, R1 is 101, R2 is 102, and so on.
```
set protocols isis source-packet-routing srgb start-label 10001
set protocols isis source-packet-routing srgb index-range 5000
set protocols isis source-packet-routing node-segment ipv4-index 101
```
Here is a "gotcha": naturally, you might think that to finish the label block at 15,000, we should start at 10,000. However, that is incorrect because 0 is still a label. Therefore, we need to start at 10,001 to achieve this.

Ok, let's check the results:
```
root@R1> show mpls label usage
Label space Total   Available        Applications
LSI         69609   69609  (100.00%) BGP/LDP VPLS with no-tunnel-services, BGP L3VPN with vrf-table-label
Block       199936  199936 (100.00%) BGP/LDP VPLS with tunnel-services, BGP L2VPN
Dynamic     487936  487905 (99.99% ) RSVP, LDP, PW, L3VPN, RSVP-P2MP, LDP-P2MP, MVPN, EVPN, BGP, SPRING-TE
Static      48576   48576  (100.00%) Static LSP, Static PW
```
We can see that we don't have a label block yet; we need to restart the rpd for the changes to take effect.
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
Now that our label block is defined, we need to do this on all routers in our network, except for the RR, of course.

Let's verify the IS-IS database; now our IGP distributes the label information!
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
Everything is set—label blocks are defined and segment-ids are correct!

Now, let's create a SR-TE. Personally, I don't like doing this manually; if you adopt SR-MPLS, it is time to consider a controller for TE decisions. Alternatively, you could continue with RSVP, though that is just my opinion.

To accomplish the goal without specifying SIDs, we need to enable auto-translate in the segment-list. This knob automatically translates the IP address into a SID to define the segment-list. Then, we apply the segment-list to the SR-TE.
```
set protocols source-packet-routing segment-list sl-r1 auto-translate
set protocols source-packet-routing segment-list sl-r1 1 ip-address 10.0.0.8
set protocols source-packet-routing segment-list sl-r1 1 label-type node
set protocols source-packet-routing segment-list sl-r1 2 ip-address 10.0.0.1
set protocols source-packet-routing segment-list sl-r1 2 label-type node

set protocols source-packet-routing source-routing-path to-R1 to 10.0.0.1
set protocols source-packet-routing source-routing-path to-R1 primary sl-r1
```
And voilà! This is like a LSP with an explicit-path. This configuration was made on R5; we need to replicate it similarly on R1.

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
You might notice the SID for R8 is 10109, even though it might seem more intuitive if it were 10108. However, this is how SR operates: we have the sum of Node-SID + Label Block. Since the label block starts at 10001, we have 10001 + 108 = 10109. Makes sense?

In the FIB/RIB, it looks like this:
```
root@R5> show route table inet.3 protocol spring-te 

inet.3: 20 destinations, 36 routes (8 active, 0 holddown, 16 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.1/32        *[SPRING-TE/8] 03:19:24, metric 1, metric2 16777224
                    >  to 10.200.0.19 via ge-0/0/2.0, Push 10102
                       to 10.200.0.17 via ae0.0, Push 10102, Push 10109, Push 10108(top)
```
Our SR-TE was computed successfully!

Now let's establish another connection with SR-TE, this time between R2 and R5. On R2, we will configure it the same way we just did, with the segment-list translating IP addresses. However, on R5, we will do things a bit differently. We will specify the SIDs of the path and forward the traffic to R2 through a specific interface of the LAG between R1 and R2.

First, we need to create a static label entry on R1. This way, R1 can forward the traffic only through ge-0/0/0 in the LAG. The task requires us to ensure that R1 uses a static label range of 40,000–45,000.

So, we create the static-label-range, then create a static label entry, specifying the next-hop address, the output interface, and the label action. R1 acts as the penultimate hop and will perform penultimate-hop-popping.
```
set protocols mpls label-range static-label-range 40000 45000

set protocols mpls static-label-switched-path via-ge-0/0/0 transit 40002 next-hop 10.200.0.1
set protocols mpls static-label-switched-path via-ge-0/0/0 transit 40002 member-interface ge-0/0/0
set protocols mpls static-label-switched-path via-ge-0/0/0 transit 40002 pop
```
With this defined, we can configure the SR-TE on R5, specifying the label stacks: first the R1 label, then the interface-specific label.
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
You can see that we don't have auto-translate this time; we are computing the SR-TE without translating IP addresses into labels.

In the FIB/RIB, it looks like this:
```
root@R5> show route table inet.3 protocol spring-te 

inet.3: 20 destinations, 36 routes (8 active, 0 holddown, 16 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32         [SPRING-TE/8] 00:04:02, metric 1, metric2 16777229
                       to 10.200.0.14 via ge-0/0/3.0, Push 40002, Push 10102(top)
                    >  to 10.200.0.19 via ge-0/0/2.0, Push 40002, Push 10102(top)
```
We have the label stack exactly as defined.

Now, let's improve our convergence during network failures using TI-LFA and node protection.
The configuration is very simple: we need to enable TI-LFA on the interfaces. For inet.3 routes, we must include the use-post-convergence-lfa knob, and to use these backup routes in inet.0 as well, we include use-source-packet-routing.

We can apply this configuration to all routers in the network:
```
set protocols isis interface all level 1 post-convergence-lfa
set protocols isis backup-spf-options use-post-convergence-lfa
set protocols isis backup-spf-options use-source-packet-routing
```
To verify if this is working properly, let's look at the outputs:
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
We can see "Post Convergence Protection: Enabled" on the interfaces.

In the RIB, we can see the backup path calculated with TI-LFA:
```
root@R1> show route table inet.3 protocol isis 

inet.3: 15 destinations, 28 routes (7 active, 0 holddown, 11 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32         [L-ISIS/14] 03:29:38, metric 5
                    >  to 10.200.0.1 via ae0.0
                       to 10.200.0.3 via ge-0/0/2.0, Push 10103, Push 10104(top)
```
Everything looks good so far.

We have reached the halfway point of the tasks, so let's do a checkpoint. Grab a coffee for the second half—let's go, you've got this!

* ~~Configure SR-MPLS on all routers, defining a block of 5,000 labels and ensuring that the final label is 15,000. Use 100 + Router Number as the Segment-ID of the router.~~
* ~~On R5, create a static SR-TE to R1 that passes through R8. On R1, create a SR-TE to R5 that passes through R7. Do not specify the SIDs explicitly in the path.~~
* ~~On R2, create a static SR-TE to R5 that passes through R4 without specifying the SIDs explicitly in the path. On R5, create a static SR-TE to R2, passing through R1 (ensure that R1 allocates the labels statically between 40,000 and 45,000) and arriving at R2 through interface ge-0/0/1 of the LAG.~~
* ~~On all routers in the network, enable TI-LFA link and node protection for routes in inet.0 and inet.3.~~
* Configure LDP tunneling in the SR-TEs between R1 and R5, and R2 and R5.
* On R3, configure a dynamic SR-TE to R4 that passes through R2. Ensure that this dynamic path is only used if there are no LDP routes to R4.
* On R3, configure microloop avoidance and ensure that the convergence paths are removed after 10 seconds.

Now, let's create another connection between LDP islands, this time using SR-TE!!!
Let's take advantage of the SR-TE created previously and add the ldp-tunneling knob. Easy peasy! I'll do this on R1 and replicate it on the others:
```
set protocols source-packet-routing source-routing-path to-R5 ldp-tunneling
```
Let's check if LDP tunneling is working:
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
It’s working! In the SPRING LSP, we can see the "LDP-tunneling enabled" line. Also, take note of "Neighbor types: auto-targeted auto-tunneled"—this indicates that LDP tunneling is functioning correctly. I think I missed this output in the RSVP article, but we found it here anyway.

Now, we need to configure a dynamic SR-TE. But what is the difference between a dynamic SR-TE and a static one?

A static SR-TE essentially tells the router which label stack to push. With a dynamic SR-TE, we tell the router to compute a path with strict or loose hops by consulting the CSPF. The router dynamically pushes the SR labels to compute a path passing through explicit hops or using a specific admin-group.

In this example, we only need to compute a path passing strictly through R2 and then arriving at R4. To ensure this SPRING-TE is only used if LDP fails, we can change the preference of SPRING-TE.
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
Now, let's look at the outputs:
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
The computed path is R3 -> R2 -> R1 -> R4. In the RIB, the SPRING-TE has a higher preference than LDP.

Finally, let's enable microloop avoidance on R3. This was a challenging topic for me at first. Microloop avoidance ensures that the entire network has updated its SPF. Consider our topology: if traffic goes from R7 to R4 and a failure occurs between R5 and R4, R5 will update its routes quickly. However, for a brief moment, R6 might not have received this info yet. R5 might forward packets to R6, and R6 might forward them back to R5 until R6 updates its own RIB.

Microloop avoidance ensures traffic is only forwarded when all routers have updated their RIBs. This is a powerful tool when combined with TI-LFA!
```
set protocols isis spf-options microloop-avoidance post-convergence-path delay 10000
```
This configuration ensures that the backup path converged by TI-LFA will be used for at least 10 seconds before another path is installed in the FIB.

Guys, we have finished our MPLS configuration!!! Now it's time to rest and prepare to configure VPN services in our network. See you next time!!!
