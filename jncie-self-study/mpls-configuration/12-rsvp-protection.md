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

