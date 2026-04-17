# Class of Service

Hello guys, I hope everyone is well today. 

This is the start of the end of the JNCIE-SP journey, abording the Class of Service topic. 

The topology you already know:
<img width="1060" height="791" alt="image" src="https://github.com/user-attachments/assets/909f3acc-3d7f-4ee0-a9ce-9dcc5f4a0886" />

In this artcile, I'll explore all CoS flow. Classifying the traffic, treating it and forwarding. Writing the flow makes all the things more simple, but this is like a puzzle, and we need to fit the pieces. 

First, let's define the queues and the schedulers, this way we define where the traffic will be, and which way the traffic will be treated. In Junos we have 8 queues by default in MX devices, so we can have 8 types of traffic in an interface, we can change the queues applied on each interface but this aren't considered a good practice. 

Another different concept in Junos is the Class of Service concept. Class of Service is considered the configuration in each device of the network, and the whole configuration in the network, is considered the Quality of Service. So here, we area configuring the Class of Service in each router, forming our QoS standard. Got it? 

So, let's go to the forwarding classes configuration. The forwarding classes are basically the queues of our router, in other words we are definiying in a specific term the queue.

We'll create 4 forwarding-classes in our network, the best-effort that is the default queue of any traffic of the network. The VPN forwarding-class, any VPN service will be in this queue. The VPN-Priority forwarding-class, a service of VPN that needs priority will be here, and finally, the Network Control forwarding-class, for the most prioritary traffic in our network, the network protocols uses this queue. See here: 
| Forwarding Class | Queue |
| ---------------- | ----- | 
| best-effort      | 0     | 
| vpn              | 1     |
| vpn-priority     | 2     | 
| nc               | 3     |

The forwarding-class is equivalent as a queue. Let's create them on all routers:
```
set class-of-service forwarding-classes queue 0 best-effort
set class-of-service forwarding-classes queue 1 vpn
set class-of-service forwarding-classes queue 2 vpn-priority
set class-of-service forwarding-classes queue 3 nc
```
Let's check the forwarding-classes existent: 
```
root@R1> show class-of-service forwarding-class
Forwarding class                       ID      Queue  Restricted queue  Fabric priority  Policing priority   SPU priority
  best-effort                           0       0          0             low                normal            low
  vpn                                   1       1          1             high               normal            low
  vpn-priority                          2       2          2             high               normal            low
  nc                                    3       3          3             high               normal            low
```
Okay, with the forwarding-classes defined, we can go to the schedulers. Schedulers are responsible to treat the traffic, defining the loss-priority of the traffic, transmit rate, buffer size reserved to this traffic, drop profiles and so on... 

Let's define the schedulers accordingly the table: 
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

You can see in the table that we have a parameter defined as drop profile, this is a method that uses a technique called Weighted Random Early Detection, or WRED. This is used to avoid the queue will be 100% full and locks. Let's define this profiles to explain the queues better. 
| Profile | Fill Level | Drop Probability | 
| - | - | - |
| low-drop | 25 | 5 |
| low-drop | 50 | 15 |
| low-drop | 75 | 40 |
| high-drop | 25 | 10 |
| high-drop | 50 | 30 |
| high-drop | 75 | 65 | 

Basically, when we have a congestion or micro-burst on the interface, basically when the routers can't forward the packets in the same way that it is arriving, the queue is filling and the drop-profile is used to discard the packets accordingly the fill level. In the low-drop, when the queue is 25% filled, 5% of this traffic will be radomly dropped, this is considered a conservative profile, the high-drop is agressive, and when we have 25% of the queue filled, 10% of the traffic will be radomly dropped. 

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
This interpolate configuration turns this in a gradative dropping, accordingly the fill level:
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
Here we have the fill level and drop proabibility side-by-side, and transforming this in a graphic, we can see it better:
<img width="680" height="790" alt="image" src="https://github.com/user-attachments/assets/9279c6ff-397f-46dd-a616-0ce2bdff03ff" />

With the images, everything looks better to understand. So, some classes will have a conservative drop-profile, and other an agresssive. 

Now, we can go back to schedulers applying this in all routers:  
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
Let's explain the queues now: 
* Best-effort: The transmit-rate specify what is the CIR, or guaranted bandwidth, and this class don't have, it will use the remainder bandwidth in the interface that is not allocated to the other queues. The buffer-size specify the size of queue on memory, and also will use the remainder. In both cases, transmit-rate and buffer-size, this queue can borrow the capacity of the other queues if it's not in use. Priority defines literally the priority of the traffic, and the drop-profile-map sets that this queue will use the high-drop profile to drop the packets with any loss-priority.

Note: Loss-priority is like a internal tag that Junos uses to classify which packet it can drops first. The loss-priority aren't marked in any field of the packet, is literally a internal feature that mark the packets when enter in the router, and is used to decide when the packet can be dropped. The loss-priority is marked on classifiers and we'll see it later. 

* VPN: The transmit-rate will have a guarantee of 20% of the interface bandwidth, and 20% of memory. The priority of this traffic will be medium-low, and here we will uses two drop-profiles, for the packets with loss-priority low, the low-drop will be used, and for the packets with high loss-priority, the high-drop will be used.

* VPN Priority: This class can be used for sensible services, like VoIP or streamings. Here we'll gurantee 10% of interface bandwidth and the buffer-size is different, when it comes to sensible services, we are talking about latency and jitter, here we are definying that the packets can stay in memory for 5ms or it will be dropped, considering the latency, the packet can be useless and no makes senses to mantain this on buffer. The priority is defined in medium-high.

* Network Control: Here we have the most prioritary traffic, we'll gurantee 5% of interface bandwidth and buffer, and the priority is high.


