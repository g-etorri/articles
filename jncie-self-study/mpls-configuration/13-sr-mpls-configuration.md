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
And voilà, this is like a LSP with an explicit-path. 

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
  LDP-tunneling enabled
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

Our SR-TE was computed with success! 
