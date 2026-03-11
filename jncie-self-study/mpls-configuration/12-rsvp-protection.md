# RSVP Protection Configuration

Hello guys! Today we're diving deep into configuring protection for our RSVP LSPs.

You already know the topology:
<img width="1026" height="793" alt="image" src="https://github.com/user-attachments/assets/e0e9790b-3451-461a-8ef7-fa01e680a242" />

Following the JNCIE-SP study style, I’ve listed our core tasks for today:
* Configure a backup path for all LSPs except: R8-R3-B, R3-R8-B, R7-R4-B, and R4-R7-B.
* Ensure LSPs R1-R6-A, R6-R1-A, R2-R5-A, and R5-R2-A have a backup path established in advance (before the primary fails).
* Prevent LSPs from doubling bandwidth reservations when a secondary path is present.
* Configure LSPs R2-R7-A, R7-R2-A, R3-R6-A, and R6-R3-A to not revert to the primary path after a failure.
* Set up Fast Reroute (FRR) for LSPs R1-R6-A, R6-R1-A, R2-R5-A, and R5-R2-A without inheriting bandwidth or admin-groups. The detour path must not exceed 5 hops.
* Apply Link Protection to LSPs: R1-R8-A, R8-R1-A, R2-R7-A, R7-R2-A, R3-R6-A, R6-R3-A, R4-R5-A, and R5-R4-A.
* Apply Node-Link Protection to LSPs: R8-R3-A, R3-R8-A, R7-R4-A, and R4-R7-A.

So, let's configure a secondary path for all the required LSPs. Here is the configuration on R1, which we need to replicate to the other routers:
```
set protocols mpls path primary
set protocols mpls path secondary
set protocols mpls label-switched-path R1-R8-A primary primary
set protocols mpls label-switched-path R1-R6-A primary primary
set protocols mpls label-switched-path R1-R8-A secondary secondary
set protocols mpls label-switched-path R1-R6-A secondary secondary
```
By creating an explicit path without defined hops, the router calculates the best CSPF path. The secondary path will ALWAYS be completely different from the primary.

Now, let's check the output to confirm if the secondary path is established correctly.
```
root@R1> show mpls lsp name R1-R6-A detail                                
Ingress LSP: 2 sessions

10.0.0.6
  From: 10.0.0.1, State: Up, ActiveRoute: 0, LSPname: R1-R6-A, LSPid: 5
  ActivePath: primary (primary)
  FastReroute desired
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Follow destination IGP metric
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  LSP Self-ping Status : Enabled
 *Primary   primary          State: Up
    Priorities: 7 7
    Bandwidth: 60Mbps
    OptimizeTimer: 28800
    SmartOptimizeTimer: 180
    Flap Count: 0
    MBB Count: 0
    Reoptimization in 14249 second(s).
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 25)
 10.200.0.1 S 10.200.0.9 S 10.200.0.20 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.200.0.1(flag=9 Label=58) 10.200.0.9(flag=1 Label=45) 10.200.0.20(Label=3)
  Secondary secondary        State: Dn
    Priorities: 7 7
    Bandwidth: 60Mbps
    OptimizeTimer: 28800
    SmartOptimizeTimer: 180
    Flap Count: 1
    MBB Count: 0
        No computed ERO.
   17 Mar 11 16:29:57.520 Clear Call
```
Here, we have both paths configured, but the secondary path is not established yet. The secondary path will only be established if the primary path fails.

Let's move on to the next task.

Here, we need to apply the standby knob to establish the secondary path alongside the primary. This improves network convergence time because if the primary path fails, the secondary path takes over immediately. A personal piece of advice: if you want a secondary path, include the standby knob. You'll maintain an extra session, but the convergence time will be significantly improved.
```
set protocols mpls label-switched-path R1-R6-A secondary secondary standby
```
Let's check the output and see the difference compared to the previous one.
```
root@R1> show mpls lsp name R1-R6-A detail    
Ingress LSP: 2 sessions

10.0.0.6
  From: 10.0.0.1, State: Up, ActiveRoute: 0, LSPname: R1-R6-A, LSPid: 5
  ActivePath: primary (primary)
  FastReroute desired
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Follow destination IGP metric
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  LSP Self-ping Status : Enabled
 *Primary   primary          State: Up
    Priorities: 7 7
    Bandwidth: 60Mbps
    OptimizeTimer: 28800
    SmartOptimizeTimer: 180
    Flap Count: 1
    MBB Count: 0
    Reoptimization in 28489 second(s).
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 25)
 10.200.0.1 S 10.200.0.7 S 10.200.0.13 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.200.0.1(Label=61) 10.200.0.7(Label=105) 10.200.0.13(Label=3)
  Standby   secondary        State: Up
    Priorities: 7 7
    Bandwidth: 60Mbps
    OptimizeTimer: 28800
    SmartOptimizeTimer: 180
    Flap Count: 1
    MBB Count: 0
    Reoptimization in 27430 second(s).
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 25)
 10.200.0.3 S 10.200.0.15 S 10.200.0.17 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.200.0.3(Label=73) 10.200.0.15(Label=64) 10.200.0.17(Label=3)
```
Now, we have the secondary path established simultaneously with the primary through a different path in our network.

Looking at the output, we can see that both paths are reserving 60Mbps of bandwidth. In other words, these LSPs will reserve 120Mbps in our network if they traverse the same path. This can be considered a waste of resources since, in the end, the LSP only needs 60Mbps.

To avoid doubling the bandwidth reservation, we can use a knob that changes the LSP reservation style. When an LSP is configured with the Fixed Filter (FF) style, secondary paths inherit the bandwidth of the primary path. By adding the adaptive knob, the reservation style changes to Shared Explicit (SE). This style allows the paths to share the same bandwidth reservation if they pass through the same link.
```
set protocols mpls label-switched-path R1-R6-A adaptive
```
I am applying this on R1, and we can replicate it across the entire network.

Simulating a scenario to validate this: I isolated R6 by disabling the interfaces to R3 and R7. On interface ae0.0, two LSPs will arrive: R3-R6-A and R1-R6-A (where we applied the adaptive knob).
```
root@R6> show rsvp session name R1-* 
Ingress RSVP: 4 sessions
Total 0 displayed, Up 0, Down 0

Egress RSVP: 4 sessions
To              From            State   Rt Style Labelin Labelout LSPname 
10.0.0.6        10.0.0.1        Up       0  1 SE       3        - R1-R6-A
10.0.0.6        10.0.0.1        Up       0  1 SE       3        - R1-R6-A
Total 2 displayed, Up 2, Down 0

root@R6> show rsvp interface ae0.0 detail 
ae0.0 Index 325, State Ena/Up
  Authentication, Aggregate, Reliable, LinkProtection
  HelloInterval 9(second)
  Address 10.200.0.17
  ActiveResv 3, PreemptionCnt 0, MaxResvTh 0bps, 0% 
  Update threshold 10.000%, UpdateThresholdValue 200Mbps
  Subscription 100%, StaticBW 2Gbps, AvailableBW 1.88Gbps, Actual 100%
  ReservedBW [0] 0bps[1] 0bps[2] 0bps[3] 0bps[4] 0bps[5] 0bps[6] 60Mbps[7] 60Mbps
...
```
We can see two RSVP sessions (one for each path), but the interface only has 60Mbps reserved. Task accomplished!

Now, let's make a "dumb" decision (haha), but we need to learn these edge cases.
"Configure LSPs R2-R7-A, R7-R2-A, R3-R6-A, and R6-R3-A to not come back to the primary path if it fails."

Logically, a secondary path is usually worse than the primary. Sometimes, the secondary isn't even the second-best path because the CSPF mechanism focuses on computing a path completely disjoint from the primary to prevent node failure. If the primary path goes down, the traffic moves to the secondary. Once the primary is available again, Junos uses a revert-timer (default is 60 seconds) to move traffic back. By setting this to 0, the LSP will stay on the secondary path until it fails.

You get the idea, right? I'm making an effort to explain this, so let me know if it's clear!

Now, let's apply this configuration:
```
set protocols mpls label-switched-path R2-R7-A revert-timer 0
```
We can see the revert-timer set to 0 in the output below:
```
root@R2> show mpls lsp name R2-R7-A detail 
Ingress LSP: 2 sessions

10.0.0.7
  From: 10.0.0.2, State: Up, ActiveRoute: 0, LSPname: R2-R7-A, LSPid: 2
  ActivePath: secondary (secondary)
  Link protection desired
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Follow destination IGP metric
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  Revert timer: 0 #<--------- HERE HERE HERE HERE
  LSP Self-ping Status : Enabled
  Primary   primary          State: Up
    Priorities: 6 6
    Bandwidth: 60Mbps
    SmartOptimizeTimer: 180
          Include Any: orange
    Flap Count: 2
    MBB Count: 0
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 25)
 10.200.0.0 S 10.200.0.5 S 10.200.0.22 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.0.0.1(flag=0x21) 10.200.0.0(flag=1 Label=65) 10.0.0.8(flag=0x21) 10.200.0.5(flag=1 Label=54) 10.0.0.7(flag=0x20) 10.200.0.22(Label=0)
 *Secondary secondary        State: Up
    Priorities: 6 6
    Bandwidth: 60Mbps
    SmartOptimizeTimer: 180
          Include Any: orange
    Flap Count: 2
    MBB Count: 0
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 25)
 10.200.0.0 S 10.200.0.5 S 10.200.0.22 S 
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          10.0.0.1(flag=0x21) 10.200.0.0(flag=1 Label=61) 10.0.0.8(flag=0x21) 10.200.0.5(flag=1 Label=50) 10.0.0.7(flag=0x20) 10.200.0.22(Label=0)
```
With the revert-timer set to 0, the path will never revert. If you prefer this behavior, I'll respect it, but I don't necessarily agree with it!

Now, let's do a checkpoint; we’ve reached the halfway point:
* ~~Configure a backup path for all LSPs except: R8-R3-B, R3-R8-B, R7-R4-B, and R4-R7-B.~~
* ~~Ensure LSPs R1-R6-A, R6-R1-A, R2-R5-A, and R5-R2-A have a backup path established in advance (before the primary fails).~~
* ~~Prevent LSPs from doubling bandwidth reservations when a secondary path is present.~~
* ~~Configure LSPs R2-R7-A, R7-R2-A, R3-R6-A, and R6-R3-A to not revert to the primary path after a failure.~~
* Set up Fast Reroute (FRR) for LSPs R1-R6-A, R6-R1-A, R2-R5-A, and R5-R2-A without inheriting bandwidth or admin-groups. The detour path must not exceed 5 hops.
* Apply Link Protection to LSPs: R1-R8-A, R8-R1-A, R2-R7-A, R7-R2-A, R3-R6-A, R6-R3-A, R4-R5-A, and R5-R4-A.
* Apply Node-Link Protection to LSPs: R8-R3-A, R3-R8-A, R7-R4-A, and R4-R7-A.

Ok, we make more than half of the tasks. 

Let's get back to our goals.
First, to achieve the remaining tasks, we need to enable link-protection on the RSVP interfaces. You can do this by specifying it on each interface or by creating a group. I prefer the latter:
```
set protocols rsvp interface ae0.0 link-protection
set protocols rsvp interface ge-0/0/2.0 link-protection
set protocols rsvp interface ge-0/0/3.0 link-protection
///
set groups rsvp protocols rsvp interface <*> link-protection
set protocols rsvp apply-groups rsvp
```

Now for the final three tasks. To configure LSPs with Fast Reroute, a 5-hop limit, and no inheritance of bandwidth or admin-groups, use the following:
```
set protocols mpls label-switched-path R1-R6-A fast-reroute hop-limit 5
set protocols mpls label-switched-path R1-R6-A fast-reroute no-include-any
```
Note: By default, Junos uses a 6-hop limit for FRR. The no-include-any ensures FRR ignores the primary path's admin-group requirements.

Checking the output:
```
root@R1> show rsvp session name R1-R6-A detail 
Ingress RSVP: 7 sessions

10.0.0.6
  From: 10.0.0.1, LSPstate: Up, ActiveRoute: 0
  LSPname: R1-R6-A, LSPpath: Primary
  LSPtype: Static Configured
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 79
  Resv style: 1 SE, Label in: -, Label out: 79
  Time left:    -, Since: Wed Mar 11 17:32:03 2026
  Tspec: rate 60Mbps size 60Mbps peak Infbps m 20 M 1500
  Port number: sender 5 receiver 47469 protocol 0
  FastReroute desired
  PATH rcvfrom: localclient 
  Adspec: sent MTU 1500
  Path MTU: received 1500
  PATH sentto: 10.200.0.3 (ge-0/0/2.0) 5 pkts
  RESV rcvfrom: 10.200.0.3 (ge-0/0/2.0) 8 pkts, Entropy label: Yes
  Explct route: 10.200.0.3 10.200.0.15 10.200.0.17 
  Record route: <self> 10.200.0.3 10.200.0.15 10.200.0.17  
    Detour is Up
    Detour Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
    Detour adspec: sent MTU 1500
    Path MTU: received 1500             
    Detour PATH sentto: 10.200.0.5 (ge-0/0/3.0) 4 pkts
    Detour RESV rcvfrom: 10.200.0.5 (ge-0/0/3.0) 5 pkts, Entropy label: Yes
    Detour Explct route: 10.200.0.5 10.200.0.18 10.200.0.17 
    Detour Record route: <self> 10.200.0.5 10.200.0.18 10.200.0.17  
    Detour Label out: 62
  Detour branch from 10.200.0.3, to skip 10.0.0.5, Up
    Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
    Adspec: received MTU 1500 
    Path MTU: received 0
    PATH rcvfrom: 10.200.0.3 (ge-0/0/2.0) 4 pkts
    Adspec: received MTU 1500 sent MTU 1500
    PATH sentto: 10.200.0.5 (ge-0/0/3.0) 1 pkts
    RESV rcvfrom: 10.200.0.5 (ge-0/0/3.0) 5 pkts, Entropy label: Yes
    Explct route: 10.200.0.5 10.200.0.18 10.200.0.17 
    Record route: 10.200.0.2 10.200.0.3 <self> 10.200.0.5 10.200.0.18
    10.200.0.17  
    Label in: 81, Label out: 62
```
We can see "FastReroute desired" and the active detours.

Now, let's move on to link-protection. Do you know the difference between link-protection and node-link-protection?

Basically, link-protection creates a bypass LSP considering the link as the Point of Local Repair (PLR). It avoids the specific failed link. Node-link-protection calculates the bypass by considering the next node as the PLR, so the bypass goes to the "next-next-hop."

Link-protection:

<img width="472" height="242" alt="image" src="https://github.com/user-attachments/assets/641f4ecd-9122-49ff-94c5-c92197648956" />

Node-link-protection:

<img width="489" height="248" alt="image" src="https://github.com/user-attachments/assets/9b85b185-e1f7-46be-b514-70c1810bdbe9" />

I draw the Bypass LSP as a "pipe" because it uses a label stack. The main LSP is encapsulated in another MPLS header: |Bypass Label|Protected LSP Label|. Usually, multiple LSPs sharing the same PLR are bundled into the same Bypass LSP.

Now, let's configure them:
```
set protocols mpls label-switched-path R1-R8-A link-protection

set protocols mpls label-switched-path R3-R8-A node-link-protection
```
The configuration is simple, but very powerful!

Checking an LSP with link-protection:
```
root@R1> show rsvp session name R1-R8-A extensive 
Ingress RSVP: 7 sessions

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
  Link protection desired
  Soft preemption desired
  Type: Link protected LSP, using Bypass->10.200.0.1
      1 Mar  6 15:14:36 Link protection up, using Bypass->10.200.0.1
  Enhanced FRR: Enabled (Downstream), LP-MP is 10.0.0.2
  PATH rcvfrom: localclient 
  Adspec: sent MTU 1500
  Path MTU: received 1500
  PATH sentto: 10.200.0.1 (ae0.0) 3 pkts
       outgoing message state: refreshing, Message ID: 157, Epoch: 5534618
  RESV rcvfrom: 10.200.0.1 (ae0.0) 6 pkts, Entropy label: Yes
       incoming message handle: R-46/6, Message ID: 184, Epoch: 5534611
  Explct route: 10.200.0.1 10.200.0.9 10.200.0.23 
  Record route: <self> 10.0.0.2 (node-id) 10.200.0.1 10.0.0.7 (node-id)
  10.200.0.9 10.0.0.8 (node-id) 10.200.0.23  
      6 Mar 10 15:38:20 Record Route:  10.0.0.2(flag=0x21) 10.200.0.1(flag=1 Label=51) 10.0.0.7(flag=0x21) 10.200.0.9(flag=1 Label=41) 10.0.0.8(flag=0x20) 10.200.0.23(Label=3)
      5 Mar 10 15:38:19 Record Route:  10.0.0.2(flag=0x20) 10.200.0.1(Label=51) 10.0.0.7(flag=0x21) 10.200.0.9(flag=1 Label=41) 10.0.0.8(flag=0x20) 10.200.0.23(Label=3)
      4 Mar  6 15:14:38 Record Route:  10.0.0.2(flag=0x21) 10.200.0.1(flag=1 Label=51) 10.0.0.7(flag=0x21) 10.200.0.9(flag=1 Label=41) 10.0.0.8(flag=0x20) 10.200.0.23(Label=3)
      3 Mar  6 15:14:36 Record Route:  10.0.0.2(flag=0x20) 10.200.0.1(Label=51) 10.0.0.7(flag=0x21) 10.200.0.9(flag=1 Label=41) 10.0.0.8(flag=0x20) 10.200.0.23(Label=3)
      2 Mar  6 15:14:35 Up 
      1 Mar  6 15:14:35 Record Route:  10.0.0.2(flag=0x20) 10.200.0.1(Label=51) 10.0.0.7(flag=0x20) 10.200.0.9(Label=41) 10.0.0.8(flag=0x20) 10.200.0.23(Label=3)
  PacketType          Sent    Received
  Path                   3           0
  Resv                   0           6
  PathErr                0           0  
  ResvErr                0           0
  PathTear               0           0
  ResvTear               0           0

root@R1> show rsvp session name Bypass->10.200.0.1 detail 
Ingress RSVP: 7 sessions

10.0.0.2
  From: 10.0.0.1, LSPstate: Up, ActiveRoute: 0
  LSPname: Bypass->10.200.0.1
  LSPtype: Static Configured
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 33
  Resv style: 1 SE, Label in: -, Label out: 33
  Time left:    -, Since: Fri Mar  6 15:14:24 2026
  Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
  Port number: sender 1 receiver 17277 protocol 0
  Type: Bypass LSP
    Number of data route tunnel through: 4
    Number of RSVP session tunnel through: 0
    Number of protected LSP instances: 3
  PATH rcvfrom: localclient 
  Adspec: sent MTU 1500
  Path MTU: received 1500
  PATH sentto: 10.200.0.5 (ge-0/0/3.0) 1 pkts
  RESV rcvfrom: 10.200.0.5 (ge-0/0/3.0) 1 pkts, Entropy label: Yes
  Explct route: 10.200.0.5 10.200.0.22 10.200.0.8 
  Record route: <self> 10.200.0.5 10.200.0.22 10.200.0.8  
      4 Mar  6 15:14:24 Up              
      3 Mar  6 15:14:24 Record Route:  10.200.0.5(Label=33) 10.200.0.22(Label=34) 10.200.0.8(Label=3)
      2 Mar  6 15:14:24 CSPF: computation result accepted
      1 Mar  6 15:14:22 Originate Call
```
R1 is protecting the link to R2 by encapsulating the traffic in the Bypass->10.200.0.1 LSP.

Now, checking node-link-protection:
```
root@R3> show rsvp session name R3-R8-A extensive    
Ingress RSVP: 6 sessions

10.0.0.8
  From: 10.0.0.3, LSPstate: Up, ActiveRoute: 0
  LSPname: R3-R8-A, LSPpath: Primary
  LSPtype: Static Configured
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 76
  Resv style: 1 SE, Label in: -, Label out: 76
  Time left:    -, Since: Wed Mar 11 18:28:29 2026
  Tspec: rate 60Mbps size 60Mbps peak Infbps m 20 M 1500
  Port number: sender 2 receiver 43667 protocol 0
  Node/Link protection desired
  Type: Node/Link protected LSP, using Bypass->10.200.0.6->10.200.0.0
      3 Mar 11 18:28:33 Node protection up, using Bypass->10.200.0.6->10.200.0.0
      2 Mar 11 18:28:32 New bypass Bypass->10.200.0.6
      1 Mar 11 18:28:31 New bypass Bypass->10.200.0.6->10.200.0.0
  Enhanced FRR: Enabled (Downstream), NP-MP is 10.0.0.1
  PATH rcvfrom: localclient 
  Adspec: sent MTU 1500
  Path MTU: received 1500
  PATH sentto: 10.200.0.6 (ge-0/0/3.0) 2 pkts
       outgoing message state: refreshing, Message ID: 340, Epoch: 5534713
  RESV rcvfrom: 10.200.0.6 (ge-0/0/3.0) 4 pkts, Entropy label: Yes
       incoming message handle: R-128/4, Message ID: 329, Epoch: 5534611
  Explct route: 10.200.0.6 10.200.0.0 10.200.0.5 
  Record route: <self> 10.0.0.2 (node-id) 10.200.0.6 10.0.0.1 (node-id)
  10.200.0.0 10.0.0.8 (node-id) 10.200.0.5  
      4 Mar 11 18:28:32 Record Route:  10.0.0.2(flag=0x29) 10.200.0.6(flag=9 Label=76) 10.0.0.1(flag=0x21) 10.200.0.0(flag=1 Label=85) 10.0.0.8(flag=0x20) 10.200.0.5(Label=3)
      3 Mar 11 18:28:31 Record Route:  10.0.0.2(flag=0x21) 10.200.0.6(flag=1 Label=76) 10.0.0.1(flag=0x21) 10.200.0.0(flag=1 Label=85) 10.0.0.8(flag=0x20) 10.200.0.5(Label=3)
      2 Mar 11 18:28:29 Up 
      1 Mar 11 18:28:29 Record Route:  10.0.0.2(flag=0x20) 10.200.0.6(Label=76) 10.0.0.1(flag=0x20) 10.200.0.0(Label=85) 10.0.0.8(flag=0x20) 10.200.0.5(Label=3)
  PacketType          Sent    Received
  Path                   2           0
  Resv                   0           4
  PathErr                0           0
  ResvErr                0           0
  PathTear               0           0
  ResvTear               0           0

root@R3> show rsvp session name Bypass->10.200.0.6->10.200.0.0 extensive 
Ingress RSVP: 6 sessions

10.0.0.1
  From: 10.0.0.3, LSPstate: Up, ActiveRoute: 0
  LSPname: Bypass->10.200.0.6->10.200.0.0
  LSPtype: Static Configured
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 89
  Resv style: 1 SE, Label in: -, Label out: 89
  Time left:    -, Since: Wed Mar 11 18:28:33 2026
  Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
  Port number: sender 1 receiver 20300 protocol 0
  Type: Bypass LSP
    Number of data route tunnel through: 2
    Number of RSVP session tunnel through: 0
    Number of protected LSP instances: 1
  Enhanced FRR: Enabled (Downstream)
  PATH rcvfrom: localclient 
  Adspec: sent MTU 1500
  Path MTU: received 1500
  PATH sentto: 10.200.0.11 (ge-0/0/4.0) 1 pkts
       outgoing message state: refreshing, Message ID: 338, Epoch: 5534713
  RESV rcvfrom: 10.200.0.11 (ge-0/0/4.0) 1 pkts, Entropy label: Yes
       incoming message handle: R-129/1, Message ID: 377, Epoch: 5534632
  Explct route: 10.200.0.11 10.200.0.2 
  Record route: <self> 10.200.0.11 10.200.0.2  
      4 Mar 11 18:28:33 Up
      3 Mar 11 18:28:33 Record Route:  10.200.0.11(Label=89) 10.200.0.2(Label=3)
      2 Mar 11 18:28:33 CSPF: computation result accepted
      1 Mar 11 18:28:31 Originate Call
      2 Mar 11 18:28:33 Up 
      1 Mar 11 18:28:33 Record Route:  10.200.0.11(Label=89) 10.200.0.2(Label=3)
  PacketType          Sent    Received
  Path                   1           0
  Resv                   0           1
  PathErr                0           0
  ResvErr                0           0
  PathTear               0           0
  ResvTear               0           0
```
Here is the difference in practice: the PLR is the next node. The Bypass LSP is established to R1, bypassing R2 entirely.

And that’s it! We have finished the RSVP configuration.

See you in the next chapter where we explore SR-MPLS, and after that, we move on to MPLS VPNs!


