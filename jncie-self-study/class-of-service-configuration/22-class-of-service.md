# Class of Service

Hello guys, I hope everyone is doing well today.

This is the beginning of the end of our JNCIE-SP journey, tackling the Class of Service topic.

Here is the topology you already know:
<img width="1060" height="791" alt="image" src="https://github.com/user-attachments/assets/909f3acc-3d7f-4ee0-a9ce-9dcc5f4a0886" />

In this article, I'll explore the entire CoS flow: classifying the traffic, treating it, and forwarding it. Writing out the flow makes things simpler, but it's like a puzzle, and we need to fit the pieces together.

First, let's define the queues and the schedulers; this way, we define where the traffic will go and how it will be treated. In Junos, we have 8 queues by default on MX devices, meaning we can have 8 types of traffic on a single interface. We can change the number of queues applied to each interface, but this isn't considered good practice.

Another unique concept in Junos is the difference between Class of Service and Quality of Service. Class of Service refers to the configuration on each individual device, while the overarching configuration across the entire network is considered Quality of Service (QoS). So here, we are configuring the CoS on each router to form our network-wide QoS standard. Got it?

So, let's move on to the forwarding classes configuration. The forwarding classes are essentially the queues of our router; in other words, we are defining the queue in specific terms.

We'll create 4 forwarding classes in our network: the ```best-effort``` class, which is the default queue for any general traffic; the ```vpn``` forwarding class, where any standard VPN service will sit; the ```vpn-priority``` forwarding class, for VPN services that require priority; and finally, the ```nc``` (Network Control) forwarding class, for the highest priority traffic in our network (network protocols use this queue). See here:
| Forwarding Class | Queue |
| ---------------- | ----- | 
| best-effort      | 0     | 
| vpn              | 1     |
| vpn-priority     | 2     | 
| nc               | 3     |

The forwarding class is equivalent to a queue. Let's create them on all routers:
```
set class-of-service forwarding-classes queue 0 best-effort
set class-of-service forwarding-classes queue 1 vpn
set class-of-service forwarding-classes queue 2 vpn-priority
set class-of-service forwarding-classes queue 3 nc
```
Let's check the existing forwarding classes:
```
root@R1> show class-of-service forwarding-class
Forwarding class                       ID      Queue  Restricted queue  Fabric priority  Policing priority   SPU priority
  best-effort                           0       0          0             low                normal            low
  vpn                                   1       1          1             high               normal            low
  vpn-priority                          2       2          2             high               normal            low
  nc                                    3       3          3             high               normal            low
```
Okay, with the forwarding classes defined, we can move on to the schedulers. Schedulers are responsible for treating the traffic—defining the loss priority, transmit rate, buffer size reserved for this traffic, drop profiles, and so on.

Let's define the schedulers according to the table:
| Scheduler     | Parameter            | Value       |
| ------------- | -------------------- | ----------- |
| be-sc-q0      | Priority             | low         |
| be-sc-q0      | Transmit Rate        | remainder   |
| be-sc-q0      | Buffer Size          | remainder   |
| be-sc-q0      | Drop profile LP any  | high-drop   |
| vpn-sc-q1     | Priority             | medium-low  |
| vpn-sc-q1     | Transmit Rate        | 20%         |
| vpn-sc-q1     | Buffer Size          | 20%         |
| vpn-sc-q1     | Drop profile LP low  | low-drop    |
| vpn-sc-q1     | Drop profile LP high | high-drop   |
| vpn-pri-sc-q2 | Priority             | medium-high |
| vpn-pri-sc-q2 | Transmit Rate        | 10%         |
| vpn-pri-sc-q2 | Buffer Size          | 5 msec      |
| nc-sc-q3      | Priority             | high        |
| nc-sc-q3      | Transmit Rate        | 5%          |
| nc-sc-q3      | Buffer Size          | 5%          |

You can see in the table that we have a parameter defined as "drop profile". This is a method that uses a technique called Weighted Random Early Detection, or WRED. This is used to prevent the queue from becoming 100% full and locking up. Let's define these profiles to explain the queues better.
| Profile | Fill Level | Drop Probability | 
| - | - | - |
| low-drop | 25 | 5 |
| low-drop | 50 | 15 |
| low-drop | 75 | 40 |
| high-drop | 25 | 10 |
| high-drop | 50 | 30 |
| high-drop | 75 | 65 | 

Basically, when we have congestion or a micro-burst on the interface—meaning the router can't forward the packets as fast as they are arriving—the queue starts filling up, and the drop profile is used to discard packets according to the fill level. In the low-drop profile, when the queue is 25% full, 5% of this traffic will be randomly dropped; this is considered a conservative profile. The high-drop profile is aggressive, so when the queue is 25% full, 10% of the traffic will be randomly dropped.

Let's configure this on all routers:
```
set class-of-service drop-profiles low-drop interpolate fill-level 25
set class-of-service drop-profiles low-drop interpolate fill-level 50
set class-of-service drop-profiles low-drop interpolate fill-level 75
set class-of-service drop-profiles low-drop interpolate drop-probability 5
set class-of-service drop-profiles low-drop interpolate drop-probability 15
set class-of-service drop-profiles low-drop interpolate drop-probability 40

set class-of-service drop-profiles high-drop interpolate fill-level 25
set class-of-service drop-profiles high-drop interpolate fill-level 50
set class-of-service drop-profiles high-drop interpolate fill-level 75
set class-of-service drop-profiles high-drop interpolate drop-probability 10
set class-of-service drop-profiles high-drop interpolate drop-probability 30
set class-of-service drop-profiles high-drop interpolate drop-probability 65
```
This ```interpolate``` configuration turns this into a gradual dropping mechanism, adjusting smoothly according to the fill level:
```
root@R8> show class-of-service drop-profile
Drop profile: <default-drop-profile>, Type: discrete, Index: 1
  Fill level    Drop probability
         100                 100
Drop profile: high-drop, Type: interpolated, Index: 48162
  Fill level    Drop probability
           0                   0
           1                   0
           2                   0
           4                   1
           5                   2
           6                   2
           8                   3
          10                   4
          12                   4
          14                   5
          15                   6
          16                   6
          18                   7
          20                   8
          22                   8
          24                   9
          25                  10
          26                  10
          28                  12
          30                  14
          32                  15
          34                  17
          35                  18
          36                  18
          38                  20
          40                  22
          42                  23
          44                  25
          45                  26
          46                  26
          48                  28
          49                  29
          51                  31
          52                  32
          54                  35
          55                  37
          56                  38
          58                  41
          60                  44
          62                  46
          64                  49
          65                  51
          66                  52
          68                  55
          70                  58
          72                  60
          74                  63
          75                  65
          76                  66
          78                  69
          80                  72
          82                  74
          84                  77
          85                  79
          86                  80
          88                  83
          90                  86
          92                  88
          94                  91
          95                  93
          96                  94
          98                  97
          99                  98
         100                 100
Drop profile: low-drop, Type: interpolated, Index: 59912
  Fill level    Drop probability
           0                   0
           1                   0
           2                   0
           4                   0
           5                   1
           6                   1
           8                   1
          10                   2
          12                   2
          14                   2
          15                   3
          16                   3
          18                   3
          20                   4
          22                   4
          24                   4
          25                   5
          26                   5
          28                   6
          30                   7
          32                   7
          34                   8
          35                   9
          36                   9
          38                  10
          40                  11
          42                  11
          44                  12
          45                  13
          46                  13
          48                  14
          49                  14
          51                  16
          52                  17
          54                  19
          55                  20
          56                  21
          58                  23
          60                  25
          62                  27
          64                  29
          65                  30
          66                  31
          68                  33
          70                  35
          72                  37
          74                  39
          75                  40
          76                  42
          78                  47
          80                  52
          82                  56
          84                  61
          85                  64
          86                  66
          88                  71
          90                  76
          92                  80
          94                  85
          95                  88
          96                  90
          98                  95
          99                  97
         100                 100
```
Here we have the fill level and drop probability side-by-side, and by transforming this into a graph, we can visualize it better:
<img width="680" height="790" alt="image" src="https://github.com/user-attachments/assets/9279c6ff-397f-46dd-a616-0ce2bdff03ff" />

With the images, everything is much easier to understand! So, some classes will have a conservative drop profile, and others will have an aggressive one.

Now, we can go back to our schedulers and apply this across all routers:
```
set class-of-service schedulers be-sc-q0 transmit-rate remainder
set class-of-service schedulers be-sc-q0 buffer-size remainder
set class-of-service schedulers be-sc-q0 priority low
set class-of-service schedulers be-sc-q0 drop-profile-map loss-priority any protocol any drop-profile high-drop

set class-of-service schedulers vpn-sc-q1 transmit-rate percent 20
set class-of-service schedulers vpn-sc-q1 buffer-size percent 20
set class-of-service schedulers vpn-sc-q1 priority medium-low
set class-of-service schedulers vpn-sc-q1 drop-profile-map loss-priority low protocol any drop-profile low-drop
set class-of-service schedulers vpn-sc-q1 drop-profile-map loss-priority high protocol any drop-profile high-drop

set class-of-service schedulers vpn-pri-sc-q2 transmit-rate percent 10
set class-of-service schedulers vpn-pri-sc-q2 buffer-size temporal 5k
set class-of-service schedulers vpn-pri-sc-q2 priority medium-high

set class-of-service schedulers nc-sc-q3 transmit-rate percent 5
set class-of-service schedulers nc-sc-q3 buffer-size percent 5
set class-of-service schedulers nc-sc-q3 priority high
```
Let's break down the queues now:
* Best-effort: The ```transmit-rate``` specify what is the CIR, or guaranted bandwidth, and this class don't have, it will use the remainder bandwidth in the interface that is not allocated to the other queues. The ```buffer-size``` specify the size of queue on memory, and also will use the remainder. In both cases, transmit-rate and buffer-size, this queue can borrow the capacity of the other queues if it's not in use. ```Priority``` defines literally the priority of the traffic, and the ```drop-profile-map``` sets that this queue will use the ```high-drop``` profile to drop the packets with any loss-priority.

Note: Loss priority is like an internal tag that Junos uses to classify which packets it can drop first. The loss priority isn't explicitly marked in any field of the actual packet; it is purely an internal feature that marks packets as they enter the router, helping the router decide when a packet can be dropped. The loss priority is set by classifiers, which we'll look at later.

* VPN: This class has a guarantee of 20% of the interface bandwidth and 20% of the buffer memory. The priority of this traffic is ```medium-low```, and here we use two drop profiles: for packets with a low loss priority, ```low-drop``` will be used. For packets with a high loss priority, ```high-drop``` will be used.

* VPN Priority: This class is meant for sensitive services, like VoIP or real-time streaming. Here we guarantee 10% of the interface bandwidth, but the buffer size is handled differently. When dealing with sensitive services, we are primarily concerned with latency and jitter. Here, we are defining that packets can stay in the memory buffer for a maximum of 5ms before being dropped. If latency is too high, the packet becomes useless, so it makes no sense to keep it in the buffer. The priority is set to ```medium-high```.

* Network Control: Here we have the highest priority traffic. We guarantee 5% of the interface bandwidth and buffer, and the priority is ```high```.

Now, to apply these schedulers to our forwarding classes, we need to use scheduler-maps. Scheduler maps are essentially responsible for linking a forwarding class to a specific scheduler.
```
set class-of-service scheduler-maps bkb-interfaces forwarding-class best-effort scheduler be-sc-q0
set class-of-service scheduler-maps bkb-interfaces forwarding-class nc scheduler nc-sc-q3
set class-of-service scheduler-maps bkb-interfaces forwarding-class vpn scheduler vpn-sc-q1
set class-of-service scheduler-maps bkb-interfaces forwarding-class vpn-priority scheduler vpn-pri-sc-q2
```
This way, the traffic within a forwarding class will receive the correct scheduler treatment.

Now, to apply these queues to the backbone interfaces, we execute the following:
```
set class-of-service interfaces ge-0/0/2 scheduler-map bkb-interfaces
set class-of-service interfaces ge-0/0/3 scheduler-map bkb-interfaces
set class-of-service interfaces ge-0/0/4 scheduler-map bkb-interfaces
set class-of-service interfaces ae0 scheduler-map bkb-interfaces
```
Here, I applied the configuration on R1, but we have to apply this to all routers similarly.

To verify if the configuration is applied correctly, we can check this output:
```
root@R8> show class-of-service scheduler-map bkb-interfaces
Scheduler map: bkb-interfaces, Index: 28224

  Scheduler: be-sc-q0, Forwarding class: best-effort, Index: 9240
    Transmit rate: remainder, Rate Limit: none, Buffer size: remainder, Buffer Limit: none, Priority: low
    Excess Priority: unspecified, Queue Depth Monitoring: disabled
    Drop profiles:
      Loss priority   Protocol    Index    Name
      Low             any         48162    high-drop
      Medium low      any         48162    high-drop
      Medium high     any         48162    high-drop
      High            any         48162    high-drop

  Scheduler: vpn-sc-q1, Forwarding class: vpn, Index: 37515
    Transmit rate: 20 percent, Rate Limit: none, Buffer size: 20 percent, Buffer Limit: none, Priority: medium-low
    Excess Priority: unspecified, Queue Depth Monitoring: disabled
    Drop profiles:
      Loss priority   Protocol    Index    Name
      Low             any         59912    low-drop
      Medium low      any             1    <default-drop-profile>
      Medium high     any             1    <default-drop-profile>
      High            any         48162    high-drop

  Scheduler: vpn-pri-sc-q2, Forwarding class: vpn-priority, Index: 44323
    Transmit rate: 10 percent, Rate Limit: none, Buffer size: 5000 us, Buffer Limit: none, Priority: medium-high
    Excess Priority: unspecified, Queue Depth Monitoring: disabled
    Drop profiles:
      Loss priority   Protocol    Index    Name
      Low             any             1    <default-drop-profile>
      Medium low      any             1    <default-drop-profile>
      Medium high     any             1    <default-drop-profile>
      High            any             1    <default-drop-profile>

  Scheduler: nc-sc-q3, Forwarding class: nc, Index: 42106
    Transmit rate: 5 percent, Rate Limit: none, Buffer size: 5 percent, Buffer Limit: none, Priority: high
    Excess Priority: unspecified, Queue Depth Monitoring: disabled
    Drop profiles:
      Loss priority   Protocol    Index    Name
      Low             any             1    <default-drop-profile>
      Medium low      any             1    <default-drop-profile>
      Medium high     any             1    <default-drop-profile>
      High            any             1    <default-drop-profile>
```
Everything looks good!!! In this output, we can see exactly which drop profiles are used for each loss-priority type. Notice that for best-effort, high-drop is used for any loss profile. For the vpn class, the custom drop profiles are used on packets with low and high loss priorities, but for medium priorities, the default drop profile is used (meaning those packets are only dropped when the queue is 100% full).

Now, we can move on to classification, policing, and marking. 
* Classification: Happens when packets enter the router. Based on certain bits in the packet (like the DSCP field), we classify the traffic.
* Policing: Involves limiting the traffic with policers. For example, we can cap a specific type of traffic at 5Mbps.
* Marking: Happens when the packet is leaving the router. The router applies rewrite rules to guarantee the DSCP value is correct or translates the DSCP value into EXP bits in an MPLS header.

Let's look at these steps in more detail.

First, let's define the values of the DSCP and EXP bits that our packets will need to be classified correctly. See this table:
| Forwarding Class | Loss Priority | Valor DSCP | Valor EXP |
| ---------------- | ------------- | ---------- | --------- |
| best-effort      | any           | 0b000000   | 0b000     |
| vpn-low          | low           | 0b001010   | 0b010     |
| vpn-high         | high          | 0b001100   | 0b011     |
| vpn-priority     | any           | 0b101110   | 0b101     |
| nc               | any           | 0b110000   | N/A       |

Notice that Network Control doesn't have an EXP bit defined, because network protocols generally don't run over the MPLS network. The exception is LDP with LDP-tunneling, but that is a special case in our lab.

We need to define these values using aliases so we can call them in the configuration:
```
set class-of-service code-point-aliases dscp best-effort 000000
set class-of-service code-point-aliases dscp nc 110000
set class-of-service code-point-aliases dscp vpn-high 001100
set class-of-service code-point-aliases dscp vpn-low 001010
set class-of-service code-point-aliases dscp vpn-priority 101110

set class-of-service code-point-aliases exp best-effort 000
set class-of-service code-point-aliases exp vpn-high 011
set class-of-service code-point-aliases exp vpn-low 010
set class-of-service code-point-aliases exp vpn-priority 101
```

Before starting with the classifier configuration, let's review the types of classifiers we have:
* Interface Classifier: The simplest classifier. It essentially classifies all traffic entering an interface. Generally, this is used on PE-CE interfaces.
* Behavior Aggregate: This classifier looks at the QoS fields on the packets (in our lab, the DSCP field and the MPLS EXP bits) to classify the traffic. Traffic is classified based on the code-point aliases we just created. Generally, this is used on CORE/Backbone interfaces.
* Multifield Classifier: This is the most granular classifier. It's basically a firewall filter rule where we can match based on source, destination, protocol, port, etc. It is applied directly to the interface, and if applied alongside a BA classifier, the MF classifier overwrites it. It's generally used on PE-CE interfaces, trunk interfaces, or any service needing deep granularity.

Okay, with that defined, our classifiers can read these fields to sort the packets. Starting with the CORE interfaces, we'll use the BA classifier:
```
set class-of-service classifiers dscp dscp-classifier forwarding-class best-effort loss-priority low code-points best-effort
set class-of-service classifiers dscp dscp-classifier forwarding-class nc loss-priority low code-points nc
set class-of-service classifiers dscp dscp-classifier forwarding-class vpn loss-priority high code-points vpn-high
set class-of-service classifiers dscp dscp-classifier forwarding-class vpn loss-priority low code-points vpn-low
set class-of-service classifiers dscp dscp-classifier forwarding-class vpn-priority loss-priority low code-points vpn-priority

set class-of-service classifiers exp exp-classifier forwarding-class best-effort loss-priority low code-points best-effort
set class-of-service classifiers exp exp-classifier forwarding-class vpn loss-priority high code-points vpn-high
set class-of-service classifiers exp exp-classifier forwarding-class vpn loss-priority low code-points vpn-low
set class-of-service classifiers exp exp-classifier forwarding-class vpn-priority loss-priority low code-points vpn-priority

set class-of-service interfaces ge-0/0/2 unit 0 classifiers dscp dscp-classifier
set class-of-service interfaces ge-0/0/2 unit 0 classifiers exp exp-classifier
set class-of-service interfaces ge-0/0/3 unit 0 classifiers dscp dscp-classifier
set class-of-service interfaces ge-0/0/3 unit 0 classifiers exp exp-classifier
set class-of-service interfaces ge-0/0/4 unit 0 classifiers dscp dscp-classifier
set class-of-service interfaces ge-0/0/4 unit 0 classifiers exp exp-classifier
set class-of-service interfaces ae0 unit 0 classifiers dscp dscp-classifier
set class-of-service interfaces ae0 unit 0 classifiers exp exp-classifier
```
Up to this point in the configuration, when packets enter through an interface, Junos will classify them by their DSCP or EXP value and bind them to a forwarding class. Once the packet is in a forwarding class, it receives treatment according to the scheduler that was mapped via the scheduler map.

To guarantee that packets are transmitted with the correct DSCP or EXP bits on the CORE interfaces, we use rewrite rules. With this configuration, the DSCP bit will be marked on network protocols and preserved as packets are routed. For MPLS services, the PE will mark the EXP bit according to the forwarding class, and on the P routers, this rule ensures the packets maintain their marked EXP bits.
```
set class-of-service interfaces ge-0/0/2 unit 0 rewrite-rules dscp dscp-rewriter
set class-of-service interfaces ge-0/0/2 unit 0 rewrite-rules exp exp-rewriter protocol mpls-inet-both
set class-of-service interfaces ge-0/0/3 unit 0 rewrite-rules dscp dscp-rewriter
set class-of-service interfaces ge-0/0/3 unit 0 rewrite-rules exp exp-rewriter protocol mpls-inet-both
set class-of-service interfaces ge-0/0/4 unit 0 rewrite-rules dscp dscp-rewriter
set class-of-service interfaces ge-0/0/4 unit 0 rewrite-rules exp exp-rewriter protocol mpls-inet-both
set class-of-service interfaces ae0 unit 0 rewrite-rules dscp dscp-rewriter
set class-of-service interfaces ae0 unit 0 rewrite-rules exp exp-rewriter protocol mpls-inet-both
```

Now, our backbone is ready to handle QoS services. To simulate a CoS deployment, we'll use Customer 3. Remembering the topology:
<img width="1617" height="1097" alt="image" src="https://github.com/user-attachments/assets/e795421d-5647-49b9-920b-30106d30e3b1" />

This customer will have two types of traffic: normal VPN traffic and priority VPN traffic. See the table:
```
| Type of Traffic | Criterion                    | Forwarding Class |
| --------------- | ---------------------------- | ---------------- |
| VPN normal      | DSCP 0B000000                | vpn              |
| VPN prioritary  | Any other DSCP value         | vpn-priority     |
```
The DSCP value here is equal to our BE configuration. So, let's head to the classifier. On the PE-CE interface, to separate traffic into two different forwarding classes based on this criteria, we need to use an MF classifier:
```
set firewall family inet filter classifier-c3 term 1 from dscp be
set firewall family inet filter classifier-c3 term 1 then forwarding-class vpn
set firewall family inet filter classifier-c3 term 1 then accept
set firewall family inet filter classifier-c3 term 2 then forwarding-class vpn-priority
set firewall family inet filter classifier-c3 term 2 then accept

set interfaces ge-0/0/8 unit 300 family inet filter input classifier-c3
```
The configuration is intuitive: when packets have the BE DSCP value, they belong to the vpn forwarding class. The second term catches everything else, placing those packets into the vpn-priority forwarding class.

With our current configuration, the customer's traffic will be properly treated during congestion or micro-bursts, giving preference to vpn-priority packets. But CoS doesn't stop there! We can also split the traffic across the two LSPs between our PEs according to the traffic type, and even limit the traffic using policers!

Our goal here is to forward normal VPN traffic through LSP B, and priority VPN traffic through LSP A. We also want to limit this traffic to the bandwidth reserved by the LSP (60Mbps). To make things interesting and learn a bit more, any excess vpn-priority traffic will be discarded, while excess normal vpn traffic will just be marked with a high loss priority.

Starting with the LSP mapping:
R3:
```
set class-of-service forwarding-policy next-hop-map lsp-map forwarding-class vpn lsp-next-hop R3-R8-B
set class-of-service forwarding-policy next-hop-map lsp-map forwarding-class vpn-priority lsp-next-hop R3-R8-A

set policy-options policy-statement load-balance-lsp term C3-LSP from route-filter 10.3.0.0/16 longer
set policy-options policy-statement load-balance-lsp term C3-LSP then cos-next-hop-map lsp-map

set routing-options forwarding-table export load-balance-lsp
```
R8:
```
set class-of-service forwarding-policy next-hop-map lsp-map forwarding-class vpn lsp-next-hop R8-R3-B
set class-of-service forwarding-policy next-hop-map lsp-map forwarding-class vpn-priority lsp-next-hop R8-R3-A

set policy-options policy-statement load-balance-lsp term C3-LSP from route-filter 10.3.0.0/16 longer
set policy-options policy-statement load-balance-lsp term C3-LSP then cos-next-hop-map lsp-map

set routing-options forwarding-table export load-balance-lsp
```
We create a ```next-hop-map lsp-map``` in the CoS configuration, and call this map in the policy applied to the forwarding table. With this, the traffic will be split accordingly.

Now, to police this traffic and enforce the rules, we need to create policers and apply them to the LSPs:
R3:
```
set firewall policer vpn-priority-policer if-exceeding bandwidth-limit 60m
set firewall policer vpn-priority-policer if-exceeding burst-size-limit 56k
set firewall policer vpn-priority-policer then discard

set firewall family any filter vpn-priority-filter term 1 then policer vpn-priority-policer
set firewall family any filter vpn-priority-filter term 1 then accept

set protocols mpls label-switched-path R3-R8-A policing filter vpn-priority-filter
--------------
set firewall policer vpn-policer if-exceeding bandwidth-limit 60m
set firewall policer vpn-policer if-exceeding burst-size-limit 56k
set firewall policer vpn-policer then loss-priority high

set firewall family any filter vpn-filter term 1 then policer vpn-policer
set firewall family any filter vpn-filter term 1 then accept

set protocols mpls label-switched-path R3-R8-B policing filter vpn-filter
```
R8:
```
set firewall policer vpn-priority-policer if-exceeding bandwidth-limit 60m
set firewall policer vpn-priority-policer if-exceeding burst-size-limit 56k
set firewall policer vpn-priority-policer then discard

set firewall family any filter vpn-priority-filter term 1 then policer vpn-priority-policer
set firewall family any filter vpn-priority-filter term 1 then accept

set protocols mpls label-switched-path R8-R3-A policing filter vpn-priority-filter
--------------
set firewall policer vpn-policer if-exceeding bandwidth-limit 60m
set firewall policer vpn-policer if-exceeding burst-size-limit 56k
set firewall policer vpn-policer then loss-priority high

set firewall family any filter vpn-filter term 1 then policer vpn-policer
set firewall family any filter vpn-filter term 1 then accept

set protocols mpls label-switched-path R8-R3-B policing filter vpn-filter
```
And... our job is finished! Now, we need to test this:
```
root@R3> show route forwarding-table destination 10.3.8.0/30
....
Routing table: VRF-C3.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
10.3.8.0/30        user     0                    indr  1048716     3
                                                 idxd      536     2
                   idx:2                         ulst  1048715     2
                              10.200.0.6        Push 17, Push 76(top)      911     2 ge-0/0/3.0
                              10.200.0.11       Push 17, Push 85, Push 89(top)      912     2 ge-0/0/4.0
                   idx:xx     10.200.0.13       Push 17, Push 111(top)      910     2 ge-0/0/2.0

root@R8> show route forwarding-table destination 10.3.3.0/30
....
Routing table: VRF-C3.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
10.3.3.0/30        user     0                    indr  1048667     3
                                                 idxd      882     2
                   idx:2                         ulst  1048666     2
                              10.200.0.4        Push 19, Push 55(top)      773     2 ge-0/0/3.0
                              10.200.0.18       Push 19, Push 54, Push 110(top)      780     2 ge-0/0/2.0
                   idx:xx     10.200.0.18       Push 19, Push 103(top)      777     2 ge-0/0/2.0
```
And from the customer's perspective:
```
[admin@CE3-1] > tool traceroute 10.3.3.2 dscp=000000
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV, STATUS
#  ADDRESS      LOSS  SENT  LAST    AVG    BEST  WORST  STD-DEV  STATUS
1  10.3.8.1     0%       5  3.6ms   131.9  3.6   544.7  209
2  10.200.0.18  0%       5  11ms    42.1   10.5  87     33.6     <MPLS:L=103,E=2 L=19,E=2,T=1>
3  10.200.0.14  0%       5  23ms    18.6   11    24     5.7      <MPLS:L=106,E=2 L=19,E=2,T=2>
4  10.3.3.1     0%       5  23.3ms  13.6   5.6   23.3   6.3
5  10.3.3.2     0%       5  7.2ms   8.9    5.9   16.2   3.8

[admin@CE3-1] > tool traceroute 10.3.3.2 dscp=000001
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV, STATUS
#  ADDRESS     LOSS  SENT  LAST    AVG    BEST  WORST  STD-DEV  STATUS
1  10.3.8.1    0%       3  4.9ms   4.2    1.9   5.8    1.7
2  10.200.0.4  0%       3  19.9ms  18.3   12    22.9   4.6      <MPLS:L=55,E=5 L=19,E=5,T=1>
3  10.200.0.1  0%       3  23.2ms  17.6   12.6  23.2   4.4      <MPLS:L=54,E=5 L=19,E=5,T=2>
4  10.3.3.1    0%       3  11.4ms  178.7  11.4  480.6  213.9
5  10.3.3.2    0%       3  10.1ms  10.6   7.5   14.2   2.8
---------
[admin@CE3-2] > tool traceroute 10.3.8.2 dscp=000000
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV, STATUS
#  ADDRESS      LOSS  SENT  LAST    AVG    BEST  WORST  STD-DEV  STATUS
1  10.3.3.1     0%       3  19.1ms  165.5  5.2   472.1  216.9
2  10.200.0.13  0%       3  91.9ms  53     11    91.9   33.1     <MPLS:L=111,E=2 L=17,E=2,T=1>
3  10.200.0.21  0%       3  17ms    56.3   10.7  141.1  60       <MPLS:L=38,E=2 L=17,E=2,T=2>
4  10.3.8.1     0%       3  14.5ms  11.4   8.2   14.5   3.2
5  10.3.8.2     0%       2  13.4ms  11.5   9.6   13.4   1.9

[admin@CE3-2] > tool traceroute 10.3.8.2 dscp=000001
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV, STATUS
#  ADDRESS     LOSS  SENT  LAST     AVG   BEST  WORST  STD-DEV  STATUS
1  10.3.3.1    0%       3  9.2ms    7.4   2.6   10.3   3.4
2  10.200.0.6  0%       3  30.7ms   20.6  11    30.7   8        <MPLS:L=76,E=5 L=17,E=5,T=1>
3  10.200.0.0  0%       3  107.6ms  43.1  10.8  107.6  45.6     <MPLS:L=85,E=5 L=17,E=5,T=2>
4  10.3.8.1    0%       3  123ms    86.2  49.4  123    36.8
5  10.3.8.2    0%       2  7ms      20.5  7     33.9   13.5
```
For traffic with the BE DSCP, the routers forward the traffic through LSPs A. For traffic with any other DSCP value, the routers forward the traffic through LSPs B!!! It's so cool, man.

With this, we have officially finished our JNCIE-SP journey. I really enjoyed writing all these articles, and I've certainly learned so much doing it. Thank you for following along and reading up to this point. See you soon!
