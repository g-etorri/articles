# RSVP Protection Configuration

Hello guys, today we'll configure protection in our RSVP LSPs!!!

You already know the topology:
<img width="1026" height="793" alt="image" src="https://github.com/user-attachments/assets/e0e9790b-3451-461a-8ef7-fa01e680a242" />

This time, I'll list all the tasks that we have to accomplish, like the JNCIE-SP study book. 
* Configure a backup path in all LSP except in R8-R3-B, R3-R8-B, R7-R4-B and R4-R7-B.
* Make sure that LSPs R1-R6-A, R6-R1-A, R2-R5-A and R5-R2-A have a backup path established in advance, before the primary fails.
* Configure all LSPs to not double the bandwidth with the secondary path.
* Configure the LSPs R2-R7-A, R7-R2-A, R3-R6-A and R6-R3-A to not come back to the primary path if fails.
* Configure the LSPs R1-R6-A, R6-R1-A, R2-R5-A and R5-R2-A to use fast-reroute, without use bandwidth or admin-group of the primary path. The devour path do not must have more than 5 hops.
* Configure the LSPs R1-R8-A, R8-R1-A, R2-R7-A, R7-R2-A, R3-R6-A, R6-R3-A, R4-R5-A and R5-R4-A to use link-protection mechanism.
* Configure the LSPs R8-R3-A, R3-R8-A, R7-R4-A and R4-R7-A to use the node-link-protection mechanism.

So, let's configure a secondary path in all LSPs that we have to do. Here the configuration in R1 and we need to replicate this to the other routers. 
```
set protocols mpls path primary
set protocols mpls path secondary
set protocols mpls label-switched-path R1-R8-A primary primary
set protocols mpls label-switched-path R1-R6-A primary primary
set protocols mpls label-switched-path R1-R8-A secondary secondary
set protocols mpls label-switched-path R1-R6-A secondary secondary
```
Creating a explicit-path without hops defined, the router calculate the best CSPF path, the secondary path ALWAYS will be totally different than primary.  

Now, let's the output to confirm if the secondary path is established correctly. 
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
Here, we have the two paths configured, but the secondary path is not established. The secondary path will be established only if the primary path fails. 

Let's go to the next task. 

Here, we need to apply the standby knob to establish the secondary path together with the primary path. This way improve the convergence time in the network, because if the primary path fails, the secondary path assumes immediatally. A personal advice here, if you want to have a secondary path, include the standby knob, you'll mantain another session established but the convergence time will be improved significantly. 
```
set protocols mpls label-switched-path R1-R6-A secondary secondary standby
```
Let's check the outputs, here we'll see the difference between the previous output. 
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
Now, we have the secondary path established simultaneously with the primary path, trough another path in our network. 

Enjoying the situation, we can see in the output that the two paths are reserving 60Mbps of bandwidth. In other words, this LSPs will reserve 120Mbps in our network if they pass by the same path. This can be considered a waste of resources, because in the final the LSPs needs only 60Mbps. 

To avoid this double of bandwidth reservation, we can use a knob that change the reservation style of LSPs. When the LSP is configured with the reservation style Fixed Filter, shown as FF in the outputs, the secondary paths inherits the bandwitdh of the primary path, in other words, reserve the same bandwitdth of the primary path. Adding the adaptive knob in the LSP, the reservation style is changed do Shared Explicit, shown as SE in the output, this style permit to reserve only the necessary bandwidth of the LSPs if the two paths pass trough the same link. 
```
set protocols mpls label-switched-path R1-R6-A adaptive
```
Here I'am setting this configuration on R1, we can replicate this in all our network. 

Simulating a scenario to validate this knob, I isolated the R6 disabling the interfaces to R3 and R7. In the interface ae0.0 will arrive two LSPs, R3-R6-A with the priority 6 and the LSP R1-R6-A that we applied the adaptive knob. 
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
We can see that we have two RSVP sessions, a session for each path, and in the interface we have only 60Mbps reserved. So, we have success in this task!!!

Now, let's make a dumb decision hahah, but we need to learn some cases, sure. 
"Configure the LSPs R2-R7-A, R7-R2-A, R3-R6-A and R6-R3-A to not come back to the primary path if fails."

Logically, the secondary path is worse than the primary, sometimes the secondary path is not the second best path, this happens because the mechanism of path computation evaluate the primary path of the LSP and focus in compute a path total different of the primary. So, If we have two physical link between two routers, not a LAG, but two physical links trough different streets. The second fiber of this path would be the second path of the CSPF, the secondary path will be computated avoiding this routers, to prevent node fail. But, if the primary fiber torns down, the LSP will converge to the secondary path, then the primary path will be computed by the secondary fiber. And this is the best path, better than the secondary path by the way, if we have the revert-timer defined, by default is defined to 60 seconds, the traffic will be converged to the primary path again, that is trough the secondary fiber, if we don't have the revert-timer, the LSP will forward the traffic trough the secondary path until the secondary path fails, then, the primary path will be active again. 

You got it, right? I made an effort to explain this, talk to me, please!!

Now, let's make this dumb configuration, no way...
```
set protocols mpls label-switched-path R2-R7-A revert-timer 0
```
We can see the revert-timer defined in the output, see below: 
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
With the revert-timer defined in 0, we'll never have a reversion of the path. So, if you prefer this situation, ok, I'll respect you, but I don't agree with this. 

Now, let's make a checkpoint of our tasks, we reached the half of them. 
* ~~Configure a backup path in all LSP except in R8-R3-B, R3-R8-B, R7-R4-B and R4-R7-B.~~
* ~~Make sure that LSPs R1-R6-A, R6-R1-A, R2-R5-A and R5-R2-A have a backup path established in advance, before the primary fails.~~
* ~~Configure all LSPs to not double the bandwidth with the secondary path.~~
* ~~Configure the LSPs R2-R7-A, R7-R2-A, R3-R6-A and R6-R3-A to not come back to the primary path if fails.~~
* Configure the LSPs R1-R6-A, R6-R1-A, R2-R5-A and R5-R2-A to use fast-reroute, without use bandwidth or admin-group of the primary path. The devour path do not must have more than 5 hops.
* Configure the LSPs R1-R8-A, R8-R1-A, R2-R7-A, R7-R2-A, R3-R6-A, R6-R3-A, R4-R5-A and R5-R4-A to use link-protection mechanism.
* Configure the LSPs R8-R3-A, R3-R8-A, R7-R4-A and R4-R7-A to use the node-link-protection mechanism.

Ok, we make more than half of the tasks. 

Let's come back to our goal. 
First, to achieve all of our goals, we need to include the link-protection in the RSVP interfaces. We can do this in two ways, specifying the link-protection in each interface, or making a group and applying in the RSVP. I prefer the last one. See below:
```
set protocols rsvp interface ae0.0 link-protection
set protocols rsvp interface ge-0/0/2.0 link-protection
set protocols rsvp interface ge-0/0/3.0 link-protection
///
set groups rsvp protocols rsvp interface <*> link-protection
set protocols rsvp apply-groups rsvp
```

Ok, with this, we can configure the final three tasks. 
To make our LSPs to have fast-reroute with 5 hops and not inherits the bandwidth and admin-groups of the LSPs we can use the follow configuration: 
```
set protocols mpls label-switched-path R1-R6-A fast-reroute hop-limit 5
set protocols mpls label-switched-path R1-R6-A fast-reroute no-include-any
```
Note: By default, Junos uses a limit of 6 hops to fast-reroute. The no-include-any makes the fast-reroute not consider the bandwitdh and admin-groups requirements. 

Now, let's check the outputs:
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
We can see the fast-reroute desired, and the detours because that. 

Now, let's follow to the link-protection LSPs. 
You know the difference of link-protection and node-link-protection? 

Basically, the link-protection makes a bypass LSPs considering the backbone link as the point of local repair, or PLR. In other words, will be created a Bypass LSP to the next hop avoiding the actual link of the LSP. And the node-link-protection calculate the bypass LSP considering the next node as PLR, so, the LSP will be computated to the next-next-hop.  

Link-protection:

<img width="472" height="242" alt="image" src="https://github.com/user-attachments/assets/641f4ecd-9122-49ff-94c5-c92197648956" />

Node-link-protection:

<img width="489" height="248" alt="image" src="https://github.com/user-attachments/assets/9b85b185-e1f7-46be-b514-70c1810bdbe9" />

I draw the Bypass LSP as a pipe, because in this cases we have a label stack. So, the main LSP is encapsulated in another MPLS header, the stack is |Bypass LSP Label|Protected LSP Label, if we have a VPN, the VPN label will be the bottom label, or the flow label. Normally, if we have some LSPs with the same PLR, all of theses LSPs are encapsulated in a Bypass LSP. 

Now, let's configure the link-protection, and node-link-protection in the LSPs designed. 
```
set protocols mpls label-switched-path R1-R8-A link-protection

set protocols mpls label-switched-path R3-R8-A node-link-protection
```
The configuration is very simple, right? It's simple but is so good in same way. 

Now, let's check a LSP with link-protection:
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
We can see that this LSP is using the Bypass->10.200.0.1 LSP! In other words, the R1 is considering the link to R2 as PLR, and to protect this LSPs, in fail case this LSP will be encapsulated in this Bypass LSP. We can see the path of the Bypass->10.200.0.1 in the output too. 

Now, let's check a LSP with the node-link-protection. 
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
Here, we can see the difference in practice, the PLR considered is the next node in the path. In simple words, the Bypass LSP is established to R1, to bypass the R2 that is considered the PLR. 

Now, we have finished the RSVP configuration!!!

See you in the next chapter to explore the SR-MPLS, before it, we can follow to the MPLS VPNs!!!


