# VPN Inter-Provider Option B Configuration

Hello guys, I hope everyone is doing well!

Today, we'll start exploring Inter-AS VPNs. In this scenario, let's simulate that we have acquired P3. So, now we need to provide L3VPN connectivity to Site 5 of Customer 3.

Here is the topology we'll use today:
<img width="1365" height="1022" alt="image" src="https://github.com/user-attachments/assets/57481a76-4d65-48a5-8c4c-5da442224ac1" />

As the title says, today we'll use the Inter-AS Option B method. For this L3VPN case, we don't need to terminate the service on the interface, but we do need the L3VPN routes to export to our external peering.

To start, we need to receive the L3VPN routes on R3. Keeping in mind that we have the route-target family configured, we need to add the ```advertise-default``` knob. This way, we'll advertise to the RR that we want to receive routes with any RT.
```
set protocols bgp group iBGP-AS65020-East family route-target advertise-default
```

Ok, now let's configure the peering with P3. Remember that we need MPLS configured on the interface to process packets with an MPLS header.
```
set interfaces ge-0/0/6 unit 302 family mpls
set protocols mpls interface ge-0/0/6.302
set protocols bgp group eBGP-AS65503-Provider3 family inet-vpn unicast
```

Let's check our adjacency and the advertisements:
```
root@R3> show bgp summary group eBGP-AS65503-Provider3
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 7 Peers: 7 Down peers: 1
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.rtarget.0
                       1          0          0          0          0          0
inet.0
                     151        142          0          0          0          0
inet6.0
                       9          5          0          0          0          0
inet.3
                       0          0          0          0          0          0
bgp.l3vpn.0
                      57         57          0          0          0          0
bgp.l3vpn-inet6.0
                       3          3          0          0          0          0
bgp.l2vpn.0
                       9          9          0          0          0          0
bgp.evpn.0
                      32         32          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.3.6            65503         13         19       0       0          11 Establ
  inet.0: 32/34/34/0
  inet6.0: 3/3/3/0
  bgp.l3vpn.0: 2/2/2/0

root@R3> show route advertising-protocol bgp 172.16.3.6 table bgp.l3vpn

bgp.l3vpn.0: 72 destinations, 72 routes (72 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  10.0.0.3:300:10.3.3.254/32
*                         Self                                    I
  10.0.0.3:12341:0.0.0.0/0
*                         Self                                    I
  10.0.0.3:12341:10.16.12.1/32
*                         Self                                    I
  10.0.0.3:12341:10.16.12.2/32
*                         Self                                    I
  10.0.0.3:12341:10.16.34.0/24
*                         Self                                    I
  10.0.0.3:12341:10.16.34.4/32
*                         Self                                    I
  10.0.0.3:12341:172.16.3.8/30
*                         Self                                    I
  10.0.0.3:100:10.1.0.2/32
*                         Self                 2                  I
  10.0.0.3:100:10.1.0.3/32
*                         Self                 3                  I
  10.0.0.3:100:10.1.3.0/30
*                         Self                                    I
  10.0.0.3:100:10.1.3.254/32
*                         Self                                    I
  10.0.0.3:100:10.1.23.0/30
*                         Self                 2                  I

root@R3> show route receive-protocol bgp 172.16.3.6 table bgp.l3vpn

bgp.l3vpn.0: 72 destinations, 72 routes (72 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  10.2.30.254:201:10.2.30.0/30
*                         172.16.3.6                              65503 I
  10.2.30.254:201:10.2.30.254/32
*                         172.16.3.6                              65503 I
```
Ok, both routers are advertising the VPN routes as expected.

Everything looks so easy! Let's check the routing table of the VRF:
```
root@R1> show route table VRF-C2-SPOKE.inet.0

VRF-C2-SPOKE.inet.0: 15 destinations, 20 routes (14 active, 0 holddown, 6 hidden)
+ = Active Route, - = Last Active, * = Both

10.2.0.3/32        *[BGP/170] 2d 23:04:15, localpref 100, from 10.0.0.0
                      AS path: 64702 I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 19
10.2.0.4/32        *[BGP/170] 2d 23:04:15, localpref 100, from 10.0.0.0
                      AS path: 64702 I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 19
10.2.0.5/32        *[BGP/170] 2d 23:04:31, localpref 100, from 10.0.0.0
                      AS path: 64702 I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 16, Push 109(top)
10.2.1.4/30        *[Direct/0] 5w5d 23:00:54
                    >  via ge-0/0/8.201
10.2.1.5/32        *[Local/0] 5w5d 23:00:54
                       Local via ge-0/0/8.201
10.2.1.253/32      *[Direct/0] 5w5d 23:02:30
                    >  via lo0.2
10.2.2.4/30        *[BGP/170] 2d 23:04:24, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 19
10.2.2.253/32      *[BGP/170] 2d 23:04:24, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 19
10.2.4.0/30        *[BGP/170] 2d 23:04:15, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 19
10.2.4.254/32      *[BGP/170] 2d 23:04:15, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 19
10.2.5.0/30        *[BGP/170] 2d 23:04:15, localpref 100, from 10.0.0.0
                      AS path: 64702 I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 19
10.2.7.0/30        *[BGP/170] 2d 23:04:31, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 16, Push 109(top)
10.2.7.254/32      *[BGP/170] 2d 23:04:31, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 16, Push 109(top)
10.2.34.0/30       *[BGP/170] 2d 23:04:15, localpref 100, from 10.0.0.0
                      AS path: 64702 I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 19
```
Oh, something is wrong here; we are not receiving the routes on R1. To start troubleshooting, let's check the route targets:
```
set routing-instances VRF-C2-SPOKE vrf-target target:65020:201
```
The RT for the spoke sites needs to be ```target:65020:201```. Now, let's check what we are receiving from P3:
```
root@R3> show route receive-protocol bgp 172.16.3.6 table bgp.l3vpn detail

bgp.l3vpn.0: 72 destinations, 72 routes (72 active, 0 holddown, 0 hidden)
* 10.2.30.254:201:10.2.30.0/30 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 10.2.30.254:201
     VPN Label: 16
     Nexthop: 172.16.3.6
     AS path: 65503 I
     Communities: target:65503:2

* 10.2.30.254:201:10.2.30.254/32 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 10.2.30.254:201
     VPN Label: 16
     Nexthop: 172.16.3.6
     AS path: 65503 I
     Communities: target:65503:2
```
Hmm, we found the problem. If you notice here, P3 is advertising the routes with a different RT. We need to normalize these advertisements to provide connectivity between the sites.

To do this, we need to create the RT communities and perform normalization through routing policies:
```
set policy-options community target-ce2-hub members target:65020:200
set policy-options community target-ce2-spoke members target:65020:201
set policy-options community target-ce2-spoke-remote members target:65503:2

set policy-options policy-statement Saida-P3 term bgpl3vpn from community target-ce2-hub
set policy-options policy-statement Saida-P3 term bgpl3vpn then community add target-ce2-spoke-remote
set policy-options policy-statement Saida-P3 term bgpl3vpn then accept

set policy-options policy-statement Entrada-P3 term bgpl3vpn from community target-ce2-spoke-remote
set policy-options policy-statement Entrada-P3 term bgpl3vpn then community add target-ce2-spoke
set policy-options policy-statement Entrada-P3 term bgpl3vpn then accept
```
Now, we can find these routes with the correct RT:
```
root@R3> show route table bgp.l3vpn.0 community target:65020:201 match-prefix 10.2.30.254*

bgp.l3vpn.0: 69 destinations, 69 routes (69 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.2.30.254:201:10.2.30.0/30
                   *[BGP/170] 00:00:27, localpref 100
                      AS path: 65503 I, validation-state: unverified
                    >  to 172.16.3.6 via ge-0/0/6.302, Push 16
10.2.30.254:201:10.2.30.254/32
                   *[BGP/170] 00:00:27, localpref 100
                      AS path: 65503 I, validation-state: unverified
                    >  to 172.16.3.6 via ge-0/0/6.302, Push 16
```

Now, let's check the VRF routing table:
```
root@R1> show route table VRF-C2-SPOKE.inet.0

VRF-C2-SPOKE.inet.0: 17 destinations, 22 routes (16 active, 0 holddown, 6 hidden)
+ = Active Route, - = Last Active, * = Both

10.2.0.3/32        *[BGP/170] 2d 23:10:44, localpref 100, from 10.0.0.0
                      AS path: 64702 I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 19
10.2.0.4/32        *[BGP/170] 2d 23:10:44, localpref 100, from 10.0.0.0
                      AS path: 64702 I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 19
10.2.0.5/32        *[BGP/170] 2d 23:11:00, localpref 100, from 10.0.0.0
                      AS path: 64702 I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 16, Push 109(top)
10.2.1.4/30        *[Direct/0] 5w5d 23:07:23
                    >  via ge-0/0/8.201
10.2.1.5/32        *[Local/0] 5w5d 23:07:23
                       Local via ge-0/0/8.201
10.2.1.253/32      *[Direct/0] 5w5d 23:08:59
                    >  via lo0.2
10.2.2.4/30        *[BGP/170] 2d 23:10:53, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 19
10.2.2.253/32      *[BGP/170] 2d 23:10:53, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 19
10.2.4.0/30        *[BGP/170] 2d 23:10:44, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 19
10.2.4.254/32      *[BGP/170] 2d 23:10:44, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 19
10.2.5.0/30        *[BGP/170] 2d 23:10:44, localpref 100, from 10.0.0.0
                      AS path: 64702 I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 19
10.2.7.0/30        *[BGP/170] 2d 23:11:00, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 16, Push 109(top)
10.2.7.254/32      *[BGP/170] 2d 23:11:00, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 16, Push 109(top)
10.2.30.0/30       *[BGP/170] 00:01:05, localpref 100, from 10.0.0.0
                      AS path: 65503 I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 650, Push 35(top)
                       to 10.200.0.3 via ge-0/0/2.0, Push 650, Push 36(top)
10.2.30.254/32     *[BGP/170] 00:01:05, localpref 100, from 10.0.0.0
                      AS path: 65503 I, validation-state: unverified
                    >  to 10.200.0.1 via ae0.0, Push 650, Push 35(top)
                       to 10.200.0.3 via ge-0/0/2.0, Push 650, Push 36(top)
10.2.34.0/30       *[BGP/170] 2d 23:10:44, localpref 100, from 10.0.0.0
                      AS path: 64702 I, validation-state: unverified
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 19
```
And everything looks great now!!!

Let's ask the customer to run some connectivity tests so we can confirm the service delivery:
```
[admin@CE2-6] > ping 10.2.0.1
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 10.2.0.1                                   56  62 4ms792us
    1 10.2.0.1                                   56  62 5ms767us
    2 10.2.0.1                                   56  62 9ms535us
    sent=3 received=3 packet-loss=0% min-rtt=4ms792us avg-rtt=6ms698us max-rtt=9ms535us

[admin@CE2-6] > ping 10.2.0.2
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 10.2.0.2                                   56  62 5ms450us
    1 10.2.0.2                                   56  62 19ms38us
    2 10.2.0.2                                   56  62 5ms543us
    sent=3 received=3 packet-loss=0% min-rtt=5ms450us avg-rtt=10ms10us max-rtt=19ms38us

[admin@CE2-6] > ping 10.2.0.3
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0                                                              no route to host
    1                                                              no route to host
    2                                                              no route to host
    sent=3 received=0 packet-loss=100%
```
This is the result. If you are a good observer, you'll notice a little detail in the topology.
```
root@R3> show route advertising-protocol bgp 172.16.3.6 table bgp.l3vpn

bgp.l3vpn.0: 69 destinations, 69 routes (69 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  10.0.0.1:200:10.2.0.1/32
*                         Self                                    64702 I
  10.0.0.1:200:10.2.0.2/32
*                         Self                                    64702 I
  10.0.0.1:200:10.2.0.254/32
*                         Self                                    I
  10.0.0.1:200:10.2.1.0/30
*                         Self                                    I
  10.0.0.1:200:10.2.1.254/32
*                         Self                                    I
  10.0.0.2:200:10.2.0.1/32
*                         Self                                    64702 I
  10.0.0.2:200:10.2.0.2/32
*                         Self                                    64702 I
  10.0.0.2:200:10.2.0.254/32
*                         Self                                    I
  10.0.0.2:200:10.2.2.0/30
*                         Self                                    I
  10.0.0.2:200:10.2.2.254/32
*                         Self                                    I
  10.0.0.3:300:10.3.3.254/32
*                         Self                                    I
  10.0.0.3:12341:0.0.0.0/0
*                         Self                                    I
  10.0.0.3:12341:10.16.12.1/32
*                         Self                                    I
  10.0.0.3:12341:10.16.12.2/32
*                         Self                                    I
  10.0.0.3:12341:10.16.34.0/24
*                         Self                                    I
  10.0.0.3:12341:10.16.34.4/32
*                         Self                                    I
  10.0.0.3:12341:172.16.3.8/30
*                         Self                                    I
  10.0.0.3:100:10.1.0.2/32
*                         Self                 2                  I
  10.0.0.3:100:10.1.0.3/32
*                         Self                 3                  I
  10.0.0.3:100:10.1.3.0/30
*                         Self                                    I
  10.0.0.3:100:10.1.3.254/32
*                         Self                                    I
  10.0.0.3:100:10.1.23.0/30
*                         Self                 2                  I
```
You can notice here that we are not advertising the default route from the HUB sites... This is happening because we need to include a simple but crucial configuration.
```
set routing-options autonomous-system 65020 loops 3
```
At the HUB site, the CEs receive the default route from a BGP session with R1 and R2. But when this route is readvertised into the VRF-HUB, our own AS is in the AS-PATH. The ```as-override``` feature won't remove the AS this time. Because of this tiny detail, we need to configure ```as-loops``` on the PEs. We can add this on all routers in the network, by the way.
```
set routing-options autonomous-system 65020 loops 3
```
Now, we should receive and advertise the default route:
```
root@R3> show route advertising-protocol bgp 172.16.3.6 table bgp.l3vpn

bgp.l3vpn.0: 71 destinations, 71 routes (71 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  10.0.0.1:200:0.0.0.0/0
*                         Self                                    64702 65020 I
  10.0.0.1:200:10.2.0.1/32
*                         Self                                    64702 I
  10.0.0.1:200:10.2.0.2/32
*                         Self                                    64702 I
  10.0.0.1:200:10.2.0.254/32
*                         Self                                    I
  10.0.0.1:200:10.2.1.0/30
*                         Self                                    I
  10.0.0.1:200:10.2.1.254/32
*                         Self                                    I
  10.0.0.2:200:0.0.0.0/0
*                         Self                                    64702 65020 I
  10.0.0.2:200:10.2.0.1/32
*                         Self                                    64702 I
  10.0.0.2:200:10.2.0.2/32
*                         Self                                    64702 I
  10.0.0.2:200:10.2.0.254/32
*                         Self                                    I
  10.0.0.2:200:10.2.2.0/30
*                         Self                                    I
  10.0.0.2:200:10.2.2.254/32
*                         Self                                    I
```
Now, everything looks perfect, literally. We can ask the customer to test again.
```
[admin@CE2-6] > ping 10.2.0.1
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 10.2.0.1                                   56  62 5ms599us
    1 10.2.0.1                                   56  62 7ms522us
    2 10.2.0.1                                   56  62 10ms975us
    sent=3 received=3 packet-loss=0% min-rtt=5ms599us avg-rtt=8ms32us max-rtt=10ms975us

[admin@CE2-6] > ping 10.2.0.3
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 10.2.0.3                                   56  59 13ms38us
    1 10.2.0.3                                   56  59 9ms267us
    sent=2 received=2 packet-loss=0% min-rtt=9ms267us avg-rtt=11ms152us max-rtt=13ms38us

[admin@CE2-6] > ping 10.2.0.1
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 10.2.0.1                                   56  62 14ms286us
    1 10.2.0.1                                   56  62 10ms32us
    2 10.2.0.1                                   56  62 5ms789us
    sent=3 received=3 packet-loss=0% min-rtt=5ms789us avg-rtt=10ms35us max-rtt=14ms286us

[admin@CE2-6] > ping 10.2.0.2
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 10.2.0.2                                   56  62 3ms977us
    1 10.2.0.2                                   56  62 4ms41us
    2 10.2.0.2                                   56  62 5ms132us
    sent=3 received=3 packet-loss=0% min-rtt=3ms977us avg-rtt=4ms383us max-rtt=5ms132us

[admin@CE2-6] > ping 10.2.0.3
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 10.2.0.3                                   56  59 35ms632us
    1 10.2.0.3                                   56  59 15ms915us
    2 10.2.0.3                                   56  59 11ms661us
    sent=3 received=3 packet-loss=0% min-rtt=11ms661us avg-rtt=21ms69us max-rtt=35ms632us

[admin@CE2-6] > ping 10.2.0.4
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 10.2.0.4                                   56  59 12ms42us
    1 10.2.0.4                                   56  59 12ms431us
    2 10.2.0.4                                   56  59 18ms248us
    sent=3 received=3 packet-loss=0% min-rtt=12ms42us avg-rtt=14ms240us max-rtt=18ms248us

[admin@CE2-6] > ping 10.2.0.5
  SEQ HOST                                     SIZE TTL TIME       STATUS
    0 10.2.0.5                                   56  59 24ms880us
    1 10.2.0.5                                   56  59 17ms929us
    2 10.2.0.5                                   56  59 14ms799us
    sent=3 received=3 packet-loss=0% min-rtt=14ms799us avg-rtt=19ms202us max-rtt=24ms880us
```
And... success!!! Service is delivered.

In the next article, we'll explore Inter-AS Option C. BGP-LU is fantastic, and on my list, BGP-LU is only second to EVPN! See you next time, homie.
