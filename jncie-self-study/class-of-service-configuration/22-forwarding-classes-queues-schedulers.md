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

You can see in the table that we have a parameter defined as drop profile, this is a method that is called Random Early Detection, or RED. This is used to avoid the queue will be 100% full and locks. Let's define this profiles to explain the queues better. 
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

Now, we can go on schedulers: 
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

