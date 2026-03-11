# RSVP Configuration

Hello guys! Today we'll connect our LDP islands using RSVP and prepare some Traffic Engineering tweaks in our lab. Let’s dive in!

You already know the topology:
<img width="1026" height="793" alt="image" src="https://github.com/user-attachments/assets/e0e9790b-3451-461a-8ef7-fa01e680a242" />

Note the colored links—these represent our admin-groups. Real-world traffic engineering is driven by constraints, so we’ll simulate those today to keep the lab realistic.

First, we enable RSVP on our backbone interfaces. We must meet the following parameters:
* Configure MD5 authentication on all backbone interfaces.
* Set a bandwidth of 333 Mbps on all interfaces, except for LAG interfaces.

It’s a simple setup. Here is the configuration for R1; you can apply similar logic to the other routers:
```
set protocols rsvp interface ae0.0 authentication-key l4b
set protocols rsvp interface ge-0/0/2.0 authentication-key l4b
set protocols rsvp interface ge-0/0/2.0 bandwidth 333m
set protocols rsvp interface ge-0/0/3.0 authentication-key l4b
set protocols rsvp interface ge-0/0/3.0 bandwidth 333m
```
We can verify that hellos are being exchanged between R1 and R2:
```
root@R1> show rsvp interface ae0.0 detail | match Hello 
  HelloInterval 9(second)
  Hello                4             2           2             1
```

Now that RSVP is running on all interfaces, let’s "color" them using admin-groups! We’ll follow this mapping:
| Router | Interface  | Admin Group  |
| ------ | ---------- | ------------ |
| R1| ge-0/0/2.0 | blue         |
| R1| ge-0/0/3.0 | orange       |
| R1| ae0.0      | blue, orange |
| R2| ge-0/0/2.0 | blue         |
| R2| ge-0/0/2.0 | orange       |
| R2| ae0.0      | blue, orange |
| R3| ge-0/0/2.0 | blue         |
| R3| ge-0/0/3.0 | orange       |
| R3| ge-0/0/4.0 | blue, orange |
| R4| ge-0/0/2.0 | blue         |
| R4| ge-0/0/3.0 | orange       |
| R4| ge-0/0/4.0 | blue, orange |
| R5| ge-0/0/2.0 | blue         |
| R5| ge-0/0/3.0 | orange       |
| R5| ae0.0      | blue, orange |
| R6| ge-0/0/2.0 | blue         |
| R6| ge-0/0/3.0 | orange       |
| R6| ae0.0      | blue. orange |
| R7| ge-0/0/2.0 | blue         |
| R7| ge-0/0/3.0 | orange       |
| R7| ge-0/0/4.0 | blue. orange |
| R8| ge-0/0/2.0 | blue         |
| R8| ge-0/0/3.0 | orange       |
| R8| ge-0/0/4.0 | blue.orange  |

Again, here is the config for R1:
```
set protocols mpls admin-groups orange 0
set protocols mpls admin-groups blue 1
set protocols mpls interface ae0.0 admin-group orange
set protocols mpls interface ae0.0 admin-group blue
set protocols mpls interface ge-0/0/3.0 admin-group orange
set protocols mpls interface ge-0/0/2.0 admin-group blue
```

With the underlay ready, let's configure our LSPs. Here is our target list:
| Ingress | Egress | LSP Name |
| ------- | ------ | -------- |
| R1      | R8     | R1-R8-A  |
| R1      | R6     | R1-R6-A  |
| R2      | R7     | R2-R7-A  |
| R2      | R5     | R2-R5-A  |
| R3      | R8     | R3-R8-A  |
| R3      | R8     | R3-R8-B  |
| R3      | R6     | R3-R6-A  |
| R4      | R7     | R4-R7-A  |
| R4      | R7     | R4-R7-B  |
| R4      | R5     | R4-R5-A  |
| R5      | R2     | R5-R2-A  |
| R5      | R4     | R5-R4-A  |
| R6      | R1     | R6-R1-A  |
| R6      | R3     | R6-R3-A  |
| R7      | R2     | R7-R2-A  |
| R7      | R4     | R7-R4-A  |
| R7      | R4     | R7-R4-B  |
| R8      | R1     | R8-R1-A  |
| R8      | R3     | R8-R3-A  |
| R8      | R3     | R8-R3-B  |

For the standard configuration, all LSPs will have BFD enabled, and we'll set the priority and hold values to the lowest priority (7 7).
To get BFD working correctly, we need to adjust our RE filter:
```
set firewall family inet filter filter-re term ALLOW-BFD from port 4784
set firewall family inet filter filter-re term ALLOW-LSP-PING from protocol udp
set firewall family inet filter filter-re term ALLOW-LSP-PING from port 3503
set firewall family inet filter filter-re term ALLOW-LSP-PING then accept
edit firewall family inet filter filter-re
insert term ALLOW-LSP-PING before term DROP-AND-COUNT
```
We added the BFD control port and permitted LSP ping, which BFD depends on. 
```
set protocols mpls label-switched-path R1-R8-A to 10.0.0.8
set protocols mpls label-switched-path R1-R8-A priority 7 7
set protocols mpls label-switched-path R1-R8-A oam bfd-liveness-detection minimum-interval 3000
set protocols mpls label-switched-path R1-R6-A to 10.0.0.6
set protocols mpls label-switched-path R1-R6-A priority 7 7
set protocols mpls label-switched-path R1-R6-A oam bfd-liveness-detection minimum-interval 3000
```
Note: By default, the hold value in Junos is 0. This means even if a new LSP has a better priority, preemption won't happen unless we explicitly define these values.

With this, we'll have the LSPs and BFD sessions established! 
```
root@R1> show mpls lsp ingress 
Ingress LSP: 2 sessions
To              From            State Rt P     ActivePath       LSPname
10.0.0.6        10.0.0.1        Up     0 *                      R1-R6-A
10.0.0.8        10.0.0.1        Up     0 *                      R1-R8-A
Total 2 displayed, Up 2, Down 0

root@R1> show bfd session 
                                                  Detect   Transmit
Address                  State     Interface      Time     Interval  Multiplier
10.0.0.6                 Up                       9.000     3.000        3   
10.0.0.8                 Up                       9.000     3.000        3  
```

Now that our LSPs are up and running, it's time to dive into some Traffic Engineering (TE) tasks. Let’s simulate a few real-world scenarios to see how we can take control of our traffic:
* Make the network invisible to external traceroutes.
* Force LSPs through specific colors in the backbone.
* Configure explicit paths to pin LSPs to a specific route.
* Signal specific bandwidth requirements for certain LSPs.
* Enable auto-bandwidth to let LSPs adjust dynamically.
* Tweak priority and hold values to give certain LSPs "VIP" status in the backbone.
* Set up soft-preemption so LSPs are re-routed before being kicked off a link.
* Configure automatic optimization for periodic path re-calculation.
* Install specific FECs into the FIB using RSVP LSPs.
* Connect LDP islands using LDP tunneling over RSVP.
* Create traffic policies to select specific LSPs based on the destination.

Ok, let's start by making our network invisible to external traceroutes. To do this, we need to stop TTL propagation on the P routers; this way, a traceroute will only show the ingress and egress routers.
```
set protocols mpls no-propagate-ttl
```
It’s that simple! If you remember the BGP chapter, I usually apply these configs after setting up MPLS. Let's see the result of no-propagate-ttl and compare it with a traceroute inside our network.  
```
root@IX-1> traceroute 201.5.1.3 
traceroute to 201.5.1.3 (201.5.1.3), 30 hops max, 40 byte packets
 1  192.168.12.1 (192.168.12.1)  17.072 ms  7.998 ms  12.781 ms
 2  10.200.0.18 (10.200.0.18)  16.269 ms  14.920 ms  16.598 ms
 3  172.16.5.6 (172.16.5.6)  13.277 ms  7.503 ms 201.5.1.3 (201.5.1.3)  9.112 ms

root@R1> traceroute 10.0.0.5 propagate-ttl 
traceroute to 10.0.0.5 (10.0.0.5), 30 hops max, 52 byte packets
 1  r4.ge0-0-2.0.jncie.lab (10.200.0.3)  6.381 ms  17.889 ms  13.717 ms
 2  r5.jncie.lab (10.0.0.5)  21.333 ms  123.781 ms  61.681 ms
```
Basically, the traceroute shows our ingress router (R1), the egress router (R5), and finally, the CE5-3. In the traceroute I ran from R1, we can see that R4 appears as a P router.

Updating our task list:
* ~~Make the network invisible to external traceroutes.~~
* Force LSPs through specific colors in the backbone.
* Configure explicit paths to pin LSPs to a specific route.
* Signal specific bandwidth requirements for certain LSPs.
* Enable auto-bandwidth to let LSPs adjust dynamically.
* Tweak priority and hold values to give certain LSPs "VIP" status in the backbone.
* Set up soft-preemption so LSPs are re-routed before being kicked off a link.
* Configure automatic optimization for periodic path re-calculation.
* Install specific FECs into the FIB using RSVP LSPs.
* Connect LDP islands using LDP tunneling over RSVP.
* Create traffic policies to select specific LSPs based on the destination.

Now, let's specify some LSPs to follow colored paths. LSPs R2-R7-A, R7-R2-A, R3-R6-A, and R6-R3-A will use the orange links, while R1-R8-A, R8-R1-A, R4-R5-A, and R5-R4-A will go through the blue ones. The configuration is straightforward:
```
set protocols mpls label-switched-path R2-R7-A admin-group include-any orange
set protocols mpls label-switched-path R3-R6-A admin-group include-any orange
set protocols mpls label-switched-path R6-R3-A admin-group include-any orange
set protocols mpls label-switched-path R7-R2-A admin-group include-any orange

set protocols mpls label-switched-path R1-R8-A admin-group include-any blue
set protocols mpls label-switched-path R8-R1-A admin-group include-any blue
set protocols mpls label-switched-path R4-R5-A admin-group include-any blue
set protocols mpls label-switched-path R5-R4-A admin-group include-any blue
```

Let's verify the path computed for a specific LSP:
```
root@R1> show mpls lsp name R1-R8-A extensive                        
Ingress LSP: 2 sessions

10.0.0.8
  From: 10.0.0.1, State: Up, ActiveRoute: 0, LSPname: R1-R8-A, LSPid: 9
  ActivePath:  (primary)
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Follow destination IGP metric
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  LSP Self-ping Status : Enabled
 *Primary                    State: Up
    Priorities: 6 6
    SmartOptimizeTimer: 180
          Include Any: blue
    Flap Count: 0
    MBB Count: 0
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 25)
 10.200.0.1 S 10.200.0.9 S 10.200.0.23 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.200.0.1(Label=157) 10.200.0.9(Label=71) 10.200.0.23(Label=3)
```
In this output, we can see the admin-group is defined and the path was computed through the blue links. Checking the topology, the path is R1 -> R2 -> R7 -> R8.

Done! Another task finished.

Updating our task list:
* ~~Make the network invisible to external traceroutes.~~
* ~~Force LSPs through specific colors in the backbone.~~
* Configure explicit paths to pin LSPs to a specific route.
* Signal specific bandwidth requirements for certain LSPs.
* Enable auto-bandwidth to let LSPs adjust dynamically.
* Tweak priority and hold values to give certain LSPs "VIP" status in the backbone.
* Set up soft-preemption so LSPs are re-routed before being kicked off a link.
* Configure automatic optimization for periodic path re-calculation.
* Install specific FECs into the FIB using RSVP LSPs.
* Connect LDP islands using LDP tunneling over RSVP.
* Create traffic policies to select specific LSPs based on the destination.

Now, let’s work with some Explicit Paths. We have a situation between R3 and R8: we need to ensure that the two LSPs on each router use two different paths. We can't use admin-groups here, and each explicit path must have exactly 3 hops.

With that in mind, I'll create two paths: North and South!
R3:
```
set protocols mpls path r8-via-north 10.0.0.2
set protocols mpls path r8-via-north 10.0.0.1
set protocols mpls path r8-via-north 10.0.0.8
set protocols mpls path r8-via-south 10.0.0.6
set protocols mpls path r8-via-south 10.0.0.7
set protocols mpls path r8-via-south 10.0.0.8
set protocols mpls label-switched-path R3-R8-A primary r8-via-north
set protocols mpls label-switched-path R3-R8-B primary r8-via-south
```
R8:
```
set protocols mpls path r3-via-north 10.0.0.1
set protocols mpls path r3-via-north 10.0.0.2
set protocols mpls path r3-via-north 10.0.0.3
set protocols mpls path r3-via-south 10.0.0.5
set protocols mpls path r3-via-south 10.0.0.4
set protocols mpls path r3-via-south 10.0.0.3
set protocols mpls label-switched-path R8-R3-A primary r3-via-north
set protocols mpls label-switched-path R8-R3-B primary r3-via-south
```
Let's check the LSP status:
```
root@R3> show mpls lsp name R3-R8-* detail 
Ingress LSP: 3 sessions

10.0.0.8
  From: 10.0.0.3, State: Up, ActiveRoute: 0, LSPname: R3-R8-A, LSPid: 5
  ActivePath: r8-via-north (primary)
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Follow destination IGP metric
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  LSP Self-ping Status : Enabled
 *Primary   r8-via-north     State: Up
    Priorities: 7 7
    SmartOptimizeTimer: 180
    Flap Count: 1
    MBB Count: 0
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 25)
 10.200.0.6 S 10.200.0.0 S 10.200.0.5 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.200.0.6(Label=161) 10.200.0.0(Label=131) 10.200.0.5(Label=3)

10.0.0.8
  From: 10.0.0.3, State: Up, ActiveRoute: 0, LSPname: R3-R8-B, LSPid: 6
  ActivePath: r8-via-south (primary)
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Follow destination IGP metric
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  LSP Self-ping Status : Enabled
 *Primary   r8-via-south     State: Up
    Priorities: 7 7
    SmartOptimizeTimer: 180
    Flap Count: 1
    MBB Count: 0
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 30)
 10.200.0.13 S 10.200.0.21 S 10.200.0.23 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.200.0.13(Label=75) 10.200.0.21(Label=74) 10.200.0.23(Label=3)
```
We can see the designed explicit path and the computed ERO exactly as expected. Success!

To make things more interesting, let’s mix Explicit Paths and Admin-Groups. Between R4 and R7, we have two pairs of LSPs. The LSPs from R7 to R4 must use orange links, and the LSPs from R4 to R7 must use blue links. On top of that, we need to ensure they use different paths to the egress routers. Let’s go!
R4:
```
set protocols mpls path r7-ep1 10.0.0.3
set protocols mpls path r7-ep2 10.0.0.1
set protocols mpls label-switched-path R4-R7-A primary r7-ep1
set protocols mpls label-switched-path R4-R7-A admin-group include-any blue
set protocols mpls label-switched-path R4-R7-B primary r7-ep2
set protocols mpls label-switched-path R4-R7-B admin-group include-any blue
```
R7:
```
set protocols mpls path r4-ep1 10.0.0.8
set protocols mpls path r4-ep2 10.0.0.2
set protocols mpls label-switched-path R7-R4-A admin-group include-any orange
set protocols mpls label-switched-path R7-R4-A primary r4-ep1
set protocols mpls label-switched-path R7-R4-B admin-group include-any orange
set protocols mpls label-switched-path R7-R4-B primary r4-ep2
```
To ensure the LSPs take different routes, we can use an explicit path with only one hop. This changes just the first hop of the LSP, and from there, the router computes the best CSPF path.

Let's check the results:
```
root@R4> show mpls lsp name R4-R7* detail       
Ingress LSP: 3 sessions

10.0.0.7
  From: 10.0.0.4, State: Up, ActiveRoute: 0, LSPname: R4-R7-A, LSPid: 5
  ActivePath: r7-ep1 (primary)
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Follow destination IGP metric
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  LSP Self-ping Status : Enabled
 *Primary   r7-ep1           State: Up
    Priorities: 7 7
    SmartOptimizeTimer: 180
          Include Any: blue
    Flap Count: 0
    MBB Count: 0
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 45)
 10.200.0.10 S 10.200.0.13 S 10.200.0.16 S 10.200.0.19 S 10.200.0.22 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.200.0.10(Label=389) 10.200.0.13(Label=76) 10.200.0.16(Label=78) 10.200.0.19(Label=65) 10.200.0.22(Label=3)

10.0.0.7                                
  From: 10.0.0.4, State: Up, ActiveRoute: 0, LSPname: R4-R7-B, LSPid: 6
  ActivePath: r7-ep2 (primary)
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Follow destination IGP metric
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  LSP Self-ping Status : Enabled
 *Primary   r7-ep2           State: Up
    Priorities: 7 7
    SmartOptimizeTimer: 180
          Include Any: blue
    Flap Count: 0
    MBB Count: 0
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 25)
 10.200.0.2 S 10.200.0.1 S 10.200.0.9 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.200.0.2(Label=132) 10.200.0.1(Label=162) 10.200.0.9(Label=3)
```
Check it out—different paths computed through the blue links! For LSP R4-R7-A, the path is R3 -> R6 -> R5 -> R8 -> R7, while R4-R7-B goes via R1 -> R2 -> R7.

Everything looks good so far!

Updating our task list:
* ~~Make the network invisible to external traceroutes.~~
* ~~Force LSPs through specific colors in the backbone.~~
* ~~Configure explicit paths to pin LSPs to a specific route.~~
* Signal specific bandwidth requirements for certain LSPs.
* Enable auto-bandwidth to let LSPs adjust dynamically.
* Tweak priority and hold values to give certain LSPs "VIP" status in the backbone.
* Set up soft-preemption so LSPs are re-routed before being kicked off a link.
* Configure automatic optimization for periodic path re-calculation.
* Install specific FECs into the FIB using RSVP LSPs.
* Connect LDP islands using LDP tunneling over RSVP.
* Create traffic policies to select specific LSPs based on the destination.

Now, let’s configure bandwidth reservation. We'll reserve 60Mbps for LSPs R1-R8-A, R8-R1-A, R4-R5-A, and R5-R4-A.
On R1, we just need to add one line and replicate it across the other routers:
```
set protocols mpls label-switched-path R1-R6-A bandwidth 60m
```
Let's check the outputs now. 
```
10.0.0.6
  From: 10.0.0.1, State: Up, ActiveRoute: 0, LSPname: R1-R6-A, LSPid: 4
  ActivePath:  (primary)
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Follow destination IGP metric
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  LSP Self-ping Status : Enabled
 *Primary                    State: Up
    Priorities: 7 7
    Bandwidth: 60Mbps
    SmartOptimizeTimer: 180
    Flap Count: 0
    MBB Count: 0
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 25)
 10.200.0.1 S 10.200.0.9 S 10.200.0.20 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.200.0.1(Label=57) 10.200.0.9(Label=44) 10.200.0.20(Label=3)
```
Looking at R1, we can see the LSP is established. How do we know the bandwidth is actually reserved? Well, if the LSP is up, it means there was enough bandwidth! But let’s double-check the transit routers:
```
root@R2> show rsvp session name R1-R6-A detail 
Transit RSVP: 13 sessions

10.0.0.6
  From: 10.0.0.1, LSPstate: Up, ActiveRoute: 0
  LSPname: R1-R6-A, LSPpath: Primary
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 44
  Resv style: 1 FF, Label in: 57, Label out: 44
  Time left: 6184, Since: Mon Mar  9 14:17:26 2026
  Tspec: rate 60Mbps size 60Mbps peak Infbps m 20 M 1500
  Port number: sender 1 receiver 47468 protocol 0
  PATH rcvfrom: 10.200.0.0 (ae0.0) 1 pkts
  Adspec: received MTU 1500 sent MTU 1500
  PATH sentto: 10.200.0.9 (ge-0/0/2.0) 1 pkts
  RESV rcvfrom: 10.200.0.9 (ge-0/0/2.0) 1 pkts, Entropy label: Yes
  Explct route: 10.200.0.9 10.200.0.20 
  Record route: 10.200.0.0 <self> 10.200.0.9 10.200.0.20  
Total 1 displayed, Up 1, Down 0

root@R2> show rsvp interface ae0.0 extensive 
ae0.0 Index 326, State Ena/Up
  Authentication, Aggregate, Reliable, LinkProtection
  HelloInterval 9(second)
  Address 10.200.0.1
  ActiveResv 10, PreemptionCnt 0, MaxResvTh 0bps, 0% 
  Update threshold 10.000%, UpdateThresholdValue 200Mbps
  Subscription 100%, Actual 100%
  bc0 = ct0, StaticBW 2Gbps
  ct0: StaticBW 2Gbps, AvailableBW 1.94Gbps
    MaxAvailableBW 2Gbps = (bc0*subscription)
    ReservedBW [0] 0bps[1] 0bps[2] 0bps[3] 0bps[4] 0bps[5] 0bps[6] 0bps[7] 60Mbps       
```
Here we see that on interface ae0.0, R2 has 2Gbps of total bandwidth, and 60Mbps has been successfully reserved by our LSP!

Updating our task list:
* ~~Become the network invisible for the external traceroutes~~
* ~~Configure LSPs to be established trough specific color in backbone~~
* ~~Configure explicit-paths to establish the LSPs trough specific path~~
* ~~Signal specific bandwidth in some LSPs~~
* ~~Configure auto-bandwitdh in some LSPs~~
* Change the priority and hold value to make some LSPs priority in the backbone
* Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.
* Configure an automatic otimization of the LSPs
* Install different FECs in the FIB using LSPs RSVP
* Connect the LDP islands with ldp-tunneling in the LSPs
* Make traffic policies using LSPs to select the path for specific destinations

Next task: Auto-bandwidth! We'll set this up for LSPs R1-R8-A, R8-R1-A, R4-R5-A, and R5-R4-A.
I’m configuring this on R1 first. We need to enable statistics collection so the router can monitor the average usage and adjust the signaled bandwidth as needed.
```
set protocols mpls statistics file auto-bw
set protocols mpls statistics auto-bandwidth
set protocols mpls label-switched-path R1-R8-A auto-bandwidth adjust-interval 172800
set protocols mpls label-switched-path R1-R8-A auto-bandwidth minimum-bandwidth 30m
set protocols mpls label-switched-path R1-R8-A auto-bandwidth maximum-bandwidth 120m
```
With this, we’re telling the router to adjust the bandwidth every 48 hours, staying between 30Mbps and 120Mbps.
```
10.0.0.8
  From: 10.0.0.1, State: Up, ActiveRoute: 0, LSPname: R1-R8-A, LSPid: 2
  ActivePath: (primary)
  Link protection desired
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Follow destination IGP metric
  Autobandwidth 
  MinBW: 30Mbps, MaxBW: 120Mbps
  AdjustTimer: 172800 secs 
  Max AvgBW util: 310.853bps, Bandwidth Adjustment in 88756 second(s).
  Overflow limit: 0, Overflow sample count: 0
  Underflow limit: 0, Underflow sample count: 280, Underflow Max AvgBW: 310.853bps
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  LSP Self-ping Status : Enabled
 *Primary             State: Up
    Priorities: 7 7
    Bandwidth: 30Mbps
    SmartOptimizeTimer: 180
          Include Any: blue
    Flap Count: 0                       
    MBB Count: 0
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 25)
 10.200.0.1 S 10.200.0.9 S 10.200.0.23 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.0.0.2(flag=0x21) 10.200.0.1(flag=1 Label=51) 10.0.0.7(flag=0x21) 10.200.0.9(flag=1 Label=41) 10.0.0.8(flag=0x20) 10.200.0.23(Label=3)
```
Upon output, we can see that there was no increase in bandwidth beyond 30 Mbps; therefore, this LSP is flagged with only 30 Mbps, and the next adjustment will be made in 88756 seconds.

Ok, autobandwitdh is also configured. We can proceed to the next task!

Updating our task list:
* ~~Become the network invisible for the external traceroutes~~
* ~~Configure LSPs to be established trough specific color in backbone~~
* ~~Configure explicit-paths to establish the LSPs trough specific path~~
* ~~Signal specific bandwidth in some LSPs~~
* ~~Configure auto-bandwitdh in some LSPs~~
* ~~Change the priority and hold value to make some LSPs priority in the backbone~~
* Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.
* Configure an automatic otimization of the LSPs
* Install different FECs in the FIB using LSPs RSVP
* Connect the LDP islands with ldp-tunneling in the LSPs
* Make traffic policies using LSPs to select the path for specific destinations

Remember how we set all LSPs to priority 7 at the start? Now it’s time to give some LSPs a "VIP" preference.
For LSPs R1-R8-A, R8-R1-A, R2-R7-A, R7-R2-A, R8-R3-A, R3-R8-A, R3-R6-A, R6-R3-A, R4-R5-A e R5-R4-A, we'll change the priority and hold values to 6. 
```
set protocols mpls label-switched-path R1-R8-A priority 6 6
```
This way, if there’s a resource crunch, LSPs with lower priority values (better priority) get preference. Since the hold value is also 6, it can preempt LSPs with a hold value of 7.

Let's see the priority and the reservation of this LSP in practice. 
```
10.0.0.8
  From: 10.0.0.1, State: Up, ActiveRoute: 0, LSPname: R1-R8-A, LSPid: 2
  ActivePath: primary (primary)
  Link protection desired
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Follow destination IGP metric
  Autobandwidth 
  MinBW: 30Mbps, MaxBW: 120Mbps
  AdjustTimer: 172800 secs 
  Max AvgBW util: 434.413bps, Bandwidth Adjustment in 83954 second(s).
  Overflow limit: 0, Overflow sample count: 0
  Underflow limit: 0, Underflow sample count: 297, Underflow Max AvgBW: 434.413bps
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  LSP Self-ping Status : Enabled
 *Primary             State: Up
    Priorities: 6 6
    Bandwidth: 30Mbps
    SmartOptimizeTimer: 180
          Include Any: blue
    Flap Count: 0                       
    MBB Count: 0
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 25)
 10.200.0.1 S 10.200.0.9 S 10.200.0.23 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.0.0.2(flag=0x21) 10.200.0.1(flag=1 Label=51) 10.0.0.7(flag=0x21) 10.200.0.9(flag=1 Label=41) 10.0.0.8(flag=0x20) 10.200.0.23(Label=3)
```
In the output we can see the priority and hold values, set to 6. 

Now, in R2 we can see the bandwitdh reservation
```
root@R2> show rsvp interface ae0.0 detail 
ae0.0 Index 326, State Ena/Up
  Authentication, Aggregate, Reliable, LinkProtection
  HelloInterval 9(second)
  Address 10.200.0.1
  ActiveResv 10, PreemptionCnt 0, MaxResvTh 0bps, 0% 
  Update threshold 10.000%, UpdateThresholdValue 200Mbps
  Subscription 100%, StaticBW 2Gbps, AvailableBW 1.91Gbps, Actual 100%
  ReservedBW [0] 0bps[1] 0bps[2] 0bps[3] 0bps[4] 0bps[5] 0bps[6] 30Mbps[7] 60Mbps
```
Note that LSP reservation occurs in a different "queue," for LSPs with priority 6. This behavior allows us flexibility in TE! It's amazing.

Ok, now we have LSPs with different priorities in our network, simulating different SLAs as a service. 

Updating our task list: 
* ~~Become the network invisible for the external traceroutes~~
* ~~Configure LSPs to be established trough specific color in backbone~~
* ~~Configure explicit-paths to establish the LSPs trough specific path~~
* ~~Signal specific bandwidth in some LSPs~~
* ~~Configure auto-bandwitdh in some LSPs~~
* ~~Change the priority and hold value to make some LSPs priority in the backbone~~
* Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.
* Configure an automatic otimization of the LSPs
* Install different FECs in the FIB using LSPs RSVP
* Connect the LDP islands with ldp-tunneling in the LSPs
* Make traffic policies using LSPs to select the path for specific destinations

Now, let's talk about Soft Preemption. Imagine a resource issue where a high-priority LSP wants to come up, but a low-priority one is already there. Normally, the low-priority one is just killed. To avoid traffic loss for our customers, soft-preemption tells the ingress router to find a new path before tearing down the old LSP.

This configuration is very simple, and we'll apply this in all LSPs of our network. 
```
set protocols mpls label-switched-path R1-R8-A soft-preemption
```
We can see if a LSP have this feature implemented, in the RSVP session:
```
10.0.0.8
  From: 10.0.0.1, LSPstate: Up, ActiveRoute: 0
  LSPname: R1-R8-A, LSPpath: Primary
  LSPtype: Static Configured
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 51
  Resv style: 1 SE, Label in: -, Label out: 51
  Time left:    -, Since: Fri Mar  6 15:14:34 2026
  Tspec: rate 30Mbps size 30Mbps peak Infbps m 20 M 1500
  Port number: sender 1 receiver 8460 protocol 0
  Soft preemption desired
```
We can see in the output with the "Soft preemption desired". 

Everything is ok!!! Let's follow for the next tasks.

Updating our task list:
* ~~Become the network invisible for the external traceroutes~~
* ~~Configure LSPs to be established trough specific color in backbone~~
* ~~Configure explicit-paths to establish the LSPs trough specific path~~
* ~~Signal specific bandwidth in some LSPs~~
* ~~Configure auto-bandwitdh in some LSPs~~
* ~~Change the priority and hold value to make some LSPs priority in the backbone~~
* ~~Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.~~
* Configure an automatic otimization of the LSPs
* Install different FECs in the FIB using LSPs RSVP
* Connect the LDP islands with ldp-tunneling in the LSPs
* Make traffic policies using LSPs to select the path for specific destinations

To keep things optimized, we can set a periodic timer: 
```
set protocols mpls optimize-timer 28800
```
In this way, our routers will optimize each LSP every 8 hours. The optimization basically consists of recalculating the LSP; if there is a better path, the LSP will be calculated based on that path; otherwise, the LSP will remain the current path.

Here we can see the output with the re-optimization configured.
```
10.0.0.6
  From: 10.0.0.1, State: Up, ActiveRoute: 0, LSPname: R1-R6-A, LSPid: 5
  ActivePath: (primary)
  FastReroute desired
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Follow destination IGP metric
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  LSP Self-ping Status : Enabled
 *Primary              State: Up
    Priorities: 7 7
    Bandwidth: 60Mbps
    OptimizeTimer: 28800
    SmartOptimizeTimer: 180
    Flap Count: 0
    MBB Count: 0
    Reoptimization in 28491 second(s).
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 25)
 10.200.0.1 S 10.200.0.9 S 10.200.0.20 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.200.0.1(flag=9 Label=58) 10.200.0.9(flag=1 Label=45) 10.200.0.20(Label=3)
```
The optimization value is fixed at 28800. This situation differs from that of LSPs with automatic bandwidth, which are optimized considering their average bandwidth. This parameter will optimize all LSPs with the actual bandwidth signaled, searching for the best path, or, without the bandwidth signaled, only the best path.

All right! Let's proceed to the next tasks.

Updating our task list:
* ~~Become the network invisible for the external traceroutes~~
* ~~Configure LSPs to be established trough specific color in backbone~~
* ~~Configure explicit-paths to establish the LSPs trough specific path~~
* ~~Signal specific bandwidth in some LSPs~~
* ~~Configure auto-bandwitdh in some LSPs~~
* ~~Change the priority and hold value to make some LSPs priority in the backbone~~
* ~~Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.~~
* ~~Configure an automatic otimization of the LSPs~~
* Install different FECs in the FIB using LSPs RSVP
* Connect the LDP islands with ldp-tunneling in the LSPs
* Make traffic policies using LSPs to select the path for specific destinations

Now, let's install specific FECs, like the IX prefix. We'll prefer the RSVP LSP over LDP (which happens naturally due to protocol preference).
```
set protocols mpls label-switched-path R5-R2-A install 192.168.12.0/24
set protocols mpls label-switched-path R6-R1-A install 192.168.12.0/24
```
Let's confirm this in the output:
```
root@R5> show route 192.168.12.1 active-path table inet.3 

inet.3: 20 destinations, 27 routes (8 active, 0 holddown, 16 hidden)
+ = Active Route, - = Last Active, * = Both

192.168.12.0/24    *[RSVP/7/1] 4d 00:09:48, metric 25
                    >  to 10.200.0.17 via ae0.0, label-switched-path R5-R2-A
```
This way, when we have a packet with destination in the IX domain, the R5 will use this LSP to forward the traffic. 

Updating our task list:
* ~~Become the network invisible for the external traceroutes~~
* ~~Configure LSPs to be established trough specific color in backbone~~
* ~~Configure explicit-paths to establish the LSPs trough specific path~~
* ~~Signal specific bandwidth in some LSPs~~
* ~~Configure auto-bandwitdh in some LSPs~~
* ~~Change the priority and hold value to make some LSPs priority in the backbone~~
* ~~Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.~~
* ~~Configure an automatic otimization of the LSPs~~
* ~~Install different FECs in the FIB using LSPs RSVP~~
* Connect the LDP islands with ldp-tunneling in the LSPs
* Make traffic policies using LSPs to select the path for specific destinations

Time to connect our LDP islands! We can achieve full connectivity by using LDP tunneling over our RSVP LSPs. Don't forget to enable LDP on lo0.0 to allow the targeted LDP sessions.
```
set protocols mpls label-switched-path R1-R8-A ldp-tunneling
set protocols mpls label-switched-path R1-R6-A ldp-tunneling

set protocols mpls label-switched-path R6-R1-A ldp-tunneling
set protocols mpls label-switched-path R8-R1-A ldp-tunneling
```
Let's see the outputs:
```
root@R1> show ldp database session 10.0.0.8 
Input label database, 10.0.0.1:0--10.0.0.8:0
Labels received: 11
  Label     Prefix
     35      10.0.0.1/32
     36      10.0.0.2/32
     26      10.0.0.3/32
     27      10.0.0.4/32
     23      10.0.0.5/32
     24      10.0.0.6/32
     22      10.0.0.7/32
      0      10.0.0.8/32
     37      192.168.12.0/24

Output label database, 10.0.0.1:0--10.0.0.8:0
Labels advertised: 10
  Label     Prefix
      0      10.0.0.1/32
     30      10.0.0.2/32
     32      10.0.0.3/32
     31      10.0.0.4/32
     33      10.0.0.5/32
     35      10.0.0.6/32                
     46      10.0.0.7/32
     53      10.0.0.8/32
      0      192.168.12.0/24
```
Here we can see that R1 and R8 are exchanging LDP mapping messages with all the FECs in our network. To visualize the LDP interconnection in action, we can see the route to R7 on R1. Theoretically, our traffic should pass through R2 and then through R7 via LDP!
```
root@R1> traceroute mpls ldp 10.0.0.7 no-resolve detail 
  Probe options: ttl 64, retries 3, wait 10, paths 16, exp 7, fanout 16

  Hop 10.200.0.1 Depth 1
    Probe status: Success
    Parent: (null)
    Return code: Return code is set per DDMT, FEC change at stack-depth 1
    Response time: 0.00 msec
    MTU: Unknown
    Multipath type: IP bitmask
      Address Range 1: 127.0.0.64 ~ 127.0.0.127
    Label Stack:
      Label 1 Value 45 Protocol LDP
    FEC-Stack-Sent: LDP
    FEC-Change-Recieved: PUSH-RSVP 

  Hop 10.200.0.0 Depth 2
    Probe status: Egress
    Parent: 10.200.0.1
    Return code: Egress-ok at stack-depth 1
    Response time: 0.00 msec
    MTU: 9216
    Multipath type: IP bitmask
      Address Range 1: 127.0.0.64 ~ 127.0.0.127
    Label Stack:                        
      Label 1 Value 61 Protocol RSVP-TE
      Label 2 Value 0 Protocol LDP
    FEC-Stack-Sent: RSVP,LDP
    FEC-Change-Recieved: POP-RSVP 

  Hop 10.200.0.0 Depth 2
    Probe status: Egress
    Parent: 10.200.0.1
    Return code: Egress-ok at stack-depth 1
    Response time: 0.00 msec
    MTU: 9216
    Multipath type: IP bitmask
      Address Range 1: 127.0.0.64 ~ 127.0.0.127
    Label Stack:
      Label 1 Value 61 Protocol RSVP-TE
      Label 2 Value 0 Protocol LDP
    FEC-Stack-Sent: LDP


  Path 1 via ae0.0 destination 127.0.0.64
```
And voilà! Our LDP interconnection is working. You can see the RSVP label being pushed and popped while the LDP label stays inside.

Updating our task list:
* ~~Become the network invisible for the external traceroutes~~
* ~~Configure LSPs to be established trough specific color in backbone~~
* ~~Configure explicit-paths to establish the LSPs trough specific path~~
* ~~Signal specific bandwidth in some LSPs~~
* ~~Configure auto-bandwitdh in some LSPs~~
* ~~Change the priority and hold value to make some LSPs priority in the backbone~~
* ~~Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.~~
* ~~Configure an automatic otimization of the LSPs~~
* ~~Install different FECs in the FIB using LSPs RSVP~~
* ~~Connect the LDP islands with ldp-tunneling in the LSPs~~
* Make traffic policies using LSPs to select the path for specific destinations

Finally, let’s look at policy-based forwarding. If R8 needs to reach P2 and P3 (connected to R3), we can split the traffic between our two LSPs:
```
set policy-options policy-statement load-balance-lsp term P2 from protocol bgp
set policy-options policy-statement load-balance-lsp term P2 from community P2
set policy-options policy-statement load-balance-lsp term P2 then install-nexthop lsp R8-R3-A
set policy-options policy-statement load-balance-lsp term P3 from protocol bgp
set policy-options policy-statement load-balance-lsp term P3 from community P3
set policy-options policy-statement load-balance-lsp term P3 then install-nexthop lsp R8-R3-B
```
We can add this terms to our load-balance policy applied in the forwarding-table, now let's take a look in the RIB/FIB:
```
root@R8> show route protocol bgp community 65020:65502 

inet.0: 221 destinations, 225 routes (218 active, 0 holddown, 5 hidden)
+ = Active Route, - = Last Active, * = Both

200.2.0.0/24       *[BGP/170] 4d 01:30:45, localpref 100, from 10.0.0.0
                      AS path: 65502 I, validation-state: unverified
                       to 10.200.0.4 via ge-0/0/3.0, label-switched-path R8-R3-A
...                               
root@R8> show route protocol bgp community 65020:65503    

inet.0: 221 destinations, 225 routes (218 active, 0 holddown, 5 hidden)
+ = Active Route, - = Last Active, * = Both

200.3.0.0/24       *[BGP/170] 4d 01:30:48, localpref 100, from 10.0.0.0
                      AS path: 65503 I, validation-state: unverified
                       to 10.200.0.18 via ge-0/0/2.0, label-switched-path R8-R3-B
...
```
And, mission accomplished!!!

We completed all the tasks.
* ~~Become the network invisible for the external traceroutes~~
* ~~Configure LSPs to be established trough specific color in backbone~~
* ~~Configure explicit-paths to establish the LSPs trough specific path~~
* ~~Signal specific bandwidth in some LSPs~~
* ~~Configure auto-bandwitdh in some LSPs~~
* ~~Change the priority and hold value to make some LSPs priority in the backbone~~
* ~~Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.~~
* ~~Configure an automatic otimization of the LSPs~~
* ~~Install different FECs in the FIB using LSPs RSVP~~
* ~~Connect the LDP islands with ldp-tunneling in the LSPs~~
* ~~Make traffic policies using LSPs to select the path for specific destinations~~

See you in the next article to configure protection on our RSVP LSPs!
