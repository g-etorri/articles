# RSVP Configuration

Hello guys, today we'll connect our LDP islands using RSVP. And we'll make some TE preparations on our LAB!!! Let's go. 

You already know the topology:
<img width="1026" height="793" alt="image" src="https://github.com/user-attachments/assets/e0e9790b-3451-461a-8ef7-fa01e680a242" />

Note that we have the links colored, this is our admin-groups! 

Today, we have so much constraints! Because, making traffic engineering IRL we'll see some constraints, and we need to learn this to do IRL. 
First, we need to configure our backbone interfaces with RSVP. In this task, we need to accomplish the follow parameters:
* Configure the MD5 authentication in all backbone interfaces. 
* Configure the bandwidth of 333 Mbps in all interface, execpt the LAG interfaces.

This is simple, here we have the configuration in R1, and you can apply the configuration similarly in the other routers: 
```
set protocols rsvp interface ae0.0 authentication-key l4b
set protocols rsvp interface ge-0/0/2.0 authentication-key l4b
set protocols rsvp interface ge-0/0/2.0 bandwidth 333m
set protocols rsvp interface ge-0/0/3.0 authentication-key l4b
set protocols rsvp interface ge-0/0/3.0 bandwidth 333m
```
We can see that hello are exchangeds between R1 and R2:
```
root@R1> show rsvp interface ae0.0 detail | match Hello 
  HelloInterval 9(second)
  Hello                4             2           2             1
```

Ok, with the RSVP configured in all interfaces, let's color the interfaces, using admin-groups! We'll do it following the table below:
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

Again, I'll do this in R1 and apply similarly in the other routers of our network:
```
set protocols mpls admin-groups orange 0
set protocols mpls admin-groups blue 1
set protocols mpls interface ae0.0 admin-group orange
set protocols mpls interface ae0.0 admin-group blue
set protocols mpls interface ge-0/0/3.0 admin-group orange
set protocols mpls interface ge-0/0/2.0 admin-group blue
```

Now, with our underlay configured, we can configure our LSPs. Here we have a list of all LSPs that we need to configure in our network:
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

First, let's make the standard configuration of the LSPs in our network. All the LSPs will have BFD working, and priority and hold with the worse value. 
To BFD working accordingly, we need to add a term in our RE filter, and modify another. 
```
set firewall family inet filter filter-re term ALLOW-BFD from port 4784
set firewall family inet filter filter-re term ALLOW-LSP-PING from protocol udp
set firewall family inet filter filter-re term ALLOW-LSP-PING from port 3503
set firewall family inet filter filter-re term ALLOW-LSP-PING then accept
edit firewall family inet filter filter-re
insert term ALLOW-LSP-PING before term DROP-AND-COUNT
```
We need to add the control port for BFD, before this we was using only the echo port. The BFD depends of the LSP ping also, so, we need to permit this in the RE filter. 
```
set protocols mpls label-switched-path R1-R8-A to 10.0.0.8
set protocols mpls label-switched-path R1-R8-A priority 7 7
set protocols mpls label-switched-path R1-R8-A oam bfd-liveness-detection minimum-interval 3000
set protocols mpls label-switched-path R1-R6-A to 10.0.0.6
set protocols mpls label-switched-path R1-R6-A priority 7 7
set protocols mpls label-switched-path R1-R6-A oam bfd-liveness-detection minimum-interval 3000
```
Note, by default the hold value on Junos will be 0, with this, even if we have a LSP with a best priority value, the preemption will not occur because the hold value is 0. We need to guarantee that our hold value is the worse, that's why the hold value need to be changed. 

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

Ok, with the LSPs established, we need to learn some tasks of traffic engineering. So, let's simulate some situations to take the actions.
* Become the network invisible for the external traceroutes
* Configure LSPs to be established trough specific color in backbone
* Configure explicit-paths to establish the LSPs trough specific path
* Signal specific bandwidth in some LSPs
* Configure auto-bandiwitdh in some LSPs
* Change the priority and hold value to make some LSPs priority in the backbone
* Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.
* Configure an automatic otimization of the LSPs
* Install different FECs in the FIB using LSPs RSVP
* Connect the LDP islands with ldp-tunneling in the LSPs
* Make traffic policies using LSPs to select the path for specific destinations

Ok, let's start becoming our network invisible to the external traceroutes. To do this, we need to not propagate the TTL in the P routers, in the traceroute we'll se only the ingress and egress routers. 
```
set protocols mpls no-propagate-ttl
```
This is simple, if you remember, in the BGP chapter I make the configurations after do the MPLS configuration, so, here I will show you the result of the no-propagate-ttl. And a result of a traceroute inside our network.  
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
Basically, the traceroute show our ingress router R1, and the egress router R5, and finally, the CE5-3. In the traceroute that I made on R1, we can see that we have de R4 as a P router. 

Updating the tasks in our network: 
* ~~Become the network invisible for the external traceroutes~~
* Configure LSPs to be established trough specific color in backbone
* Configure explicit-paths to establish the LSPs trough specific path
* Signal specific bandwidth in some LSPs
* Configure auto-bandiwitdh in some LSPs
* Change the priority and hold value to make some LSPs priority in the backbone
* Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.
* Configure an automatic otimization of the LSPs
* Install different FECs in the FIB using LSPs RSVP
* Connect the LDP islands with ldp-tunneling in the LSPs
* Make traffic policies using LSPs to select the path for specific destinations

Now, let's specify some LSPs to be established in specific colored paths. The LSPs R2-R7-A, R7-R2-A, R3-R6-A and R6-R3-A will be established trough orange links, and the LSPs R1-R8-A, R8-R1-A, R4-R5-A e R5-R4-A trough blue links. The configuration is very simple:
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

Now, let's verify a specific LSP to check what is the path computed. 
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
In this output we can see that we have the admin-group defined, and the path computed trough blue links, we can check the topology image to check, the path is R1 -> R2 -> R7 -> R8. 

Ok, this task is finished too!!!

Updating the tasks in our network: 
* ~~Become the network invisible for the external traceroutes~~
* ~~Configure LSPs to be established trough specific color in backbone~~
* Configure explicit-paths to establish the LSPs trough specific path
* Signal specific bandwidth in some LSPs
* Configure auto-bandiwitdh in some LSPs
* Change the priority and hold value to make some LSPs priority in the backbone
* Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.
* Configure an automatic otimization of the LSPs
* Install different FECs in the FIB using LSPs RSVP
* Connect the LDP islands with ldp-tunneling in the LSPs
* Make traffic policies using LSPs to select the path for specific destinations

Now, let's do some EPs. Here, we have a situation between R3 and R8, we need to guarantee that the two LSPs in each router uses two differents paths, we can't use admin-groups and each explicit-path must have only 3 hops. 

With the situation defined, I'll create two paths, path north and south! 
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
Ok, let's check the LSP now:
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
In the output we can see the explicit-path designed, and the computed ERO accordingly we have expected. So, this is a sucess case! 

Now, to make the things more interesting, let's do a mix of explicit-path and admin-groups! Between R4 and R7 we have two pair of LSPs in each router, the LSP from R7 to R4 must use orange links, and the LSPs from R4 to R7 must use the blue links, but in this case, we need to ensure that the LSPs use different paths to egress routers. Let's go.
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
To ensure that LSPs uses different paths, we can use a explicit path with only one hop, so that we are changing the first hop of the LSP only, and so on the LSPs will be computed using the best IGP path from de first hop. 

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
In the output we can see the different paths computed trough blue links! In the LSP R4-R7-A the path is R3 -> R6 -> R5-> R8 -> R7, and in the LSP R4-R7-B the path is R1 -> R2 -> R7. 

Everything looks good so far. 

Updating the tasks in our network: 
* ~~Become the network invisible for the external traceroutes~~
* ~~Configure LSPs to be established trough specific color in backbone~~
* ~~Configure explicit-paths to establish the LSPs trough specific path~~
* Signal specific bandwidth in some LSPs
* Configure auto-bandiwitdh in some LSPs
* Change the priority and hold value to make some LSPs priority in the backbone
* Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.
* Configure an automatic otimization of the LSPs
* Install different FECs in the FIB using LSPs RSVP
* Connect the LDP islands with ldp-tunneling in the LSPs
* Make traffic policies using LSPs to select the path for specific destinations

Now, let's configure the bandwitdth reserve in some LSPs, let's apply a reserve of 60Mbps on LSP R1-R8-A, R8-R1-A, R4-R5-A and R5-R4-A. 
In the R1, let's add only one line, and we can repplicate similarly on the other routers. 
```
set protocols mpls label-switched-path R1-R6-A bandwidth 60m
```
Let's check the outputs now. 

In R1, we can see that our LSP is established. 
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
Here we can see that we have the bandwidth reserved in the path correctly, but how we can know this? If the LSP is established, the path have bandwitdh sufficient hahah. Let's view in the transit routers. 
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
Here we can see that in the interface ae0.0 the R2 have 2Gbps of total bandwidth available and 60Mbps was reserved, by our LSP! 

Everything is ok! Now, we can go for the next task.

Updating the tasks in our network: 
* ~~Become the network invisible for the external traceroutes~~
* ~~Configure LSPs to be established trough specific color in backbone~~
* ~~Configure explicit-paths to establish the LSPs trough specific path~~
* ~~Signal specific bandwidth in some LSPs~~
* Configure auto-bandwitdh in some LSPs
* Change the priority and hold value to make some LSPs priority in the backbone
* Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.
* Configure an automatic otimization of the LSPs
* Install different FECs in the FIB using LSPs RSVP
* Connect the LDP islands with ldp-tunneling in the LSPs
* Make traffic policies using LSPs to select the path for specific destinations

Now, let's configure the auto-bandwitdh feature in some LSPs!!! We'll configure this on the LSPs R1-R8-A, R8-R1-A, R4-R5-A e R5-R4-A.
Here I am configuring in the R1, and we can repplicate on the other routers. For this function, we need to create a file and configure the auto-bandwidth knob, with this, the router will collect the average bandwitdh of the LSPs with auto-bandwidth configured, and will can change the bandidth signaled accordingly the necessity. 
```
set protocols mpls statistics file auto-bw
set protocols mpls statistics auto-bandwidth
set protocols mpls label-switched-path R1-R8-A auto-bandwidth adjust-interval 172800
set protocols mpls label-switched-path R1-R8-A auto-bandwidth minimum-bandwidth 30m
set protocols mpls label-switched-path R1-R8-A auto-bandwidth maximum-bandwidth 120m
```
In this configuration, we are saying to the router adjust the bandwith in each 48 hours, and the LSPs can have the minimum of 30Mbps signalled and the maximum of 120Mbps signalled. So, with time the bandwidth will vary between 30Mbps and 120Mbps accordingly the average bandwitdh collected by the router. 
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
In the output, we can see that we don't have any increase of the bandwitdh beyond the 30Mbps, so this LSPs is signalled with 30Mbps only, and the next adjust will be realized in 88756 seconds. 

Ok, autobandwitdh configured also. We can go to the next task!!

Updating the tasks in our network: 
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

Do you rememeber that before start the tasks, we defined a standard model of LSP with the priority and hold value of 7? It's time to prefer some LSPs in our network!

In the LSPs R1-R8-A, R8-R1-A, R2-R7-A, R7-R2-A, R8-R3-A, R3-R8-A, R3-R6-A, R6-R3-A, R4-R5-A e R5-R4-A we'll define the priority and hold value of 6. Again, I'll configure in R1 and you can repplicate on the other routers. 
```
set protocols mpls label-switched-path R1-R8-A priority 6 6
```
This way, if we have a issue with resources in a interface, the LSP with the minor value of priority will be preferred, and the hold value will define if the LSPs current estalblished will be preempted or not. If the value of hold is equal, the LSP established will be established until a recomputation, if the hold value of our LSP is less than the LSP established, a preemption will occur. 

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
In the output we can see the priority and hold values, defined as 6. 

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
Note that the reservation of the LSP occurs in another "queue", for the LSPs with priority 6, this behavior allows us to have flexiblity in TE!!! It's amazing. 

Ok, now we have LSPs with different priorities in our network, simulating different SLAs as a service. 

Let's follow for the tasks. 
Updating the tasks in our network: 
* ~~Become the network invisible for the external traceroutes~~
* ~~Configure LSPs to be established trough specific color in backbone~~
* ~~Configure explicit-paths to establish the LSPs trough specific path~~
* ~~Signal specific bandwidth in some LSPs~~
* ~~Configure auto-bandwitdh in some LSPs~~
* ~~Change the priority and hold value to make some LSPs priority in the backbone~~
* Configure soft-preemption so that LSPs that will be preempted, will be resignaled before removed.
* Configure an automatic otimization of the LSPs
* Configure a secondary path for the LSPs, without double the reserved bandwidth. 
* Install different FECs in the FIB using LSPs RSVP
* Connect the LDP islands with ldp-tunneling in the LSPs
* Make traffic policies using LSPs to select the path for specific destinations

Now, we have another mission, to make our network smoothly during the convergence. Thinkwith me now, if we have a resource issue in interface, where we have a LSP with a minor priority wanting to establish, and we have a LSP with the priority and hold value less than this LSP. The LSP currently established will be torned down. This could be a problem for our customers, that could have a traffic loss. To avoid this problem, we can add the soft-preemption knob in our LSPs, with this, the router that have the resource inssuficient can send a preemption pending message to the ingress router, then, the ingress router will compute the LSPs trough another path, without traffic loss. 

This configuration is very simple, and we'll apply this in all LSPs of our network. 
```
set protocols mpls label-switched-path R1-R8-A soft-preemption
```
