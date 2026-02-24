# eBGP Peering Configuration

Hello guys, today we`ll make de eBGP peering in our ASN. In this is our topology:
<img width="979" height="833" alt="image" src="https://github.com/user-attachments/assets/06333cff-6ecc-4727-aa40-53a288f4f13b" />

For this task, we will do some particularities to practice our BGP handling. 

First, we need the information to configure the interfaces, addresses and BGP peering, let`s check the table below:
| Router | Interface | IPv4 Address | IPv6 Address | VLAN | Neighbor Role | Remote AS |
| - | - | - | - | - | - | - | 
| R1 | ge-0/0/5.120 | 192.168.12.1/24 | N/A | 120 | IX | 1620 |
| R2 | ge-0/0/5.120 | 192.168.12.2/24 | N/A | 120 | IX | 1620 |
| R3 | ge-0/0/5.301 | 172.16.3.1/30 | link-local | 301 | Provider 2 | 65502 | 
| R3 | ge-0/0/5.302 | 172.16.3.5/30 | IPv4 compatible | 302 | Provider 3 | 65503 |
| R5 | ge-0/0/8.501 | 172.16.5.1/30 | fc09:c0:ffee:5::1/126 | 501 | Customer 5 | 64505 |
| R5 | ge-0/0/9.502 | 172.16.5.5/30 | fc09:c0:ffee:5::5/126 | 502 | Customer 5 | 64505 |
| R6 | ge-0/0/8.601 | 172.16.6.1/30 | fc09:c0:ffee:6::1/126 | 601 | Customer 1 | 64501 |
| R6 | ge-0/0/5.602 | 172.16.6.5/30 | N/A | 602 | Customer 2 | 64502 |
| R6 | ge-0/0/9.603 | 172.16.6.9/30 | N/A | 603 | Customer 2 | 64502 |
| R7 | ge-0/0/5.701 | 172.16.7.1/30 | fc09:c0:ffee:7::1/126 | 701 | Provider 1 | 65501 |
| R7 | ge-0/0/1.702 | 172.16.7.5/30 | fc09:c0:ffee:7::5/126 | 702 | Customer 1 | 64501 |
| R8 | ge-0/0/1.801 | 172.16.8.1/30 | fc09:c0:ffee:8::1/126 | 801 | Provider 1 | 65501 |

For all sessions, we`ll need to log the state changes of the session in the logs. 

So we need to apply the _log-updown_ in every session that we configure in our network. 

And, in every session we need to receive only the routes bigger than /24 and less than /8, otherwise, we`ll reject the routes. 

Now, I'll follow the peering cases of every router. 

In the IX environment, we have two routers, and we need to establish the session with both of them, in R1 and R2: 

The IX-1 have the IP 192.168.12.254 and the IX-2 192.168.12.252. Here I have a tip to give to you, always mark a community in the import policy, this way, we can handle the routes imported by this peering easily. 

First, we need to configure the interfaces correctly:

R1:
```
set interfaces ge-0/0/5 description to-IX-LAB
set interfaces ge-0/0/5 flexible-vlan-tagging
set interfaces ge-0/0/5 unit 120 description IX
set interfaces ge-0/0/5 unit 120 vlan-id 120
set interfaces ge-0/0/5 unit 120 family inet address 192.168.12.1/24
```
R2:
```
set interfaces ge-0/0/5 description to-IX-LAB
set interfaces ge-0/0/5 flexible-vlan-tagging
  set interfaces ge-0/0/5 unit 120 description IX
set interfaces ge-0/0/5 unit 120 vlan-id 120
set interfaces ge-0/0/5 unit 120 family inet address 192.168.12.2/24
```
Then, we can define the policy for this session, as I say before, we can start defining our communities (We'll create these communities in every router in our network):

Let's create the generic communities, the blackhole community, and the peering role communities. 
```
set policy-options community Customer members 65020:51
set policy-options community IX members 65020:1620
set policy-options community Provider members 65020:50
set policy-options community Blackhole members 6450.:666
```
In this step, we can make the blackhole community with a regex, this way we can inform the customer to export the routes with his own AS:666, and our routers will discard these routes.

And, we can configure the specific community for each peering, to do more flexible policies. 
```
set policy-options community C1 members 65020:64501
set policy-options community C2 members 65020:64502
set policy-options community C5 members 65020:64505
set policy-options community P1 members 65020:65501
set policy-options community P2 members 65020:65502
set policy-options community P3 members 65020:65503
```

Ok, with this, we can go back to the policies and peerings. 

For this situation, we'll accept all the routes between /8 and /24, and we'll add the IX community. Let's apply this policy in R1 and R2: 
```
set policy-options policy-statement Entrada-IX term 1 from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-IX term 1 then community add IX
set policy-options policy-statement Entrada-IX term 1 then accept
```
Now, we need to decide what we'll export to the IX, will be the Customers routes and our DCs' routes. 

If you remember our past articles, you know that our DCs have the follow prefixes: DC1 - 172.17.1.0/24, DC2 - 172.17.2.0/24 and DC3 - 172.17.3.0/24. To summary the routes, we can aggregate these routes on the prefix 172.17.0.0/22. 
```
set routing-options aggregate route 172.17.0.0/22 as-path aggregator 65020 10.0.0.1
set routing-options aggregate route 172.17.0.0/22 discard
```
We can apply this aggregate route in every router similarly. 

Ok, with this, we can export the customers routes and our DCs routes to IX, let's apply this on R1 and R2:
```
set policy-options policy-statement Saida-IX term Customers from community Customer
set policy-options policy-statement Saida-IX term Customers then accept
set policy-options policy-statement Saida-IX term DCs from route-filter 172.17.0.0/22 exact
set policy-options policy-statement Saida-IX term DCs then accept
set policy-options policy-statement Saida-IX then reject
```

Now, we have defined our routing policies, and we can configure the peering finally. In this situation, we want to prefer to receive the download in R1, so, this way we can use MED in this case. Let's apply the MED 10 in the R2:

R1:
```
set protocols bgp group eBGP-IX-LAB type external
set protocols bgp group eBGP-IX-LAB description eBGP-IX-LAB
set protocols bgp group eBGP-IX-LAB log-updown
set protocols bgp group eBGP-IX-LAB import Entrada-IX
set protocols bgp group eBGP-IX-LAB family inet unicast
set protocols bgp group eBGP-IX-LAB export Saida-IX
set protocols bgp group eBGP-IX-LAB peer-as 1620
set protocols bgp group eBGP-IX-LAB neighbor 192.168.12.254
set protocols bgp group eBGP-IX-LAB neighbor 192.168.12.253
```
R2:
```
set protocols bgp group eBGP-IX-LAB description eBGP-IX-LAB
set protocols bgp group eBGP-IX-LAB metric-out 10
set protocols bgp group eBGP-IX-LAB log-updown
set protocols bgp group eBGP-IX-LAB import Entrada-IX
set protocols bgp group eBGP-IX-LAB family inet unicast
set protocols bgp group eBGP-IX-LAB export Saida-IX
set protocols bgp group eBGP-IX-LAB peer-as 1620
set protocols bgp group eBGP-IX-LAB neighbor 192.168.12.254
set protocols bgp group eBGP-IX-LAB neighbor 192.168.12.253
```

Let's verify what we receive in the IX;
```
root@R2> show route receive-protocol bgp 192.168.12.253 

inet.0: 225 destinations, 277 routes (220 active, 0 holddown, 5 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  1.1.1.0/24              192.168.12.253                          1620 111 I
  8.8.8.0/24              192.168.12.253                          1620 888 I
  162.0.1.0/24            192.168.12.253                          1620 I
  162.0.2.0/24            192.168.12.253                          1620 I
  162.0.3.0/24            192.168.12.253                          1620 I
  162.0.4.0/24            192.168.12.253                          1620 I
  162.0.5.0/24            192.168.12.253                          1620 I
  162.0.6.0/24            192.168.12.253                          1620 I
  162.0.7.0/24            192.168.12.253                          1620 I
  162.0.8.0/24            192.168.12.253                          1620 I
  162.0.9.0/24            192.168.12.253                          1620 I
  162.0.10.0/24           192.168.12.253                          1620 I
  162.0.11.0/24           192.168.12.253                          1620 I
  162.0.12.0/24           192.168.12.253                          1620 I
  162.0.13.0/24           192.168.12.253                          1620 I
  162.0.14.0/24           192.168.12.253                          1620 I
  162.0.15.0/24           192.168.12.253                          1620 I
  162.0.16.0/24           192.168.12.253                          1620 I
  162.0.17.0/24           192.168.12.253                          1620 I
  162.0.18.0/24           192.168.12.253                          1620 I
  162.0.19.0/24           192.168.12.253                          1620 I
```
Ok!!! We are receiving the networks of the most famous DNSs! 1.1.1.1 and 8.8.8.8. And some others prefixes. 

And let's verify if our R2 are exporting the DC prefix with the metric 10:
```
root@R2> show route advertising-protocol bgp 192.168.12.253 172.17.0.0/22 

inet.0: 225 destinations, 277 routes (220 active, 0 holddown, 5 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 172.17.0.0/22           Self                 10                 I
```

Now, I have a idea, we can use the network 172.17.0.0/24 in our Routers, for connectivity tests only. 

Let's configure the IP in lo0 accordingly, in R1 we'll configure the 172.17.0.1 and in R2 172.17.0.2, and so on in the other routers...
```
set interfaces lo0 unit 0 family inet address 10.0.0.1/32 primary
set interfaces lo0 unit 0 family inet address 172.17.0.1/32
```
This way, we can ping the IX-1 and IX-2 from R1 and R2:
```
root@R1> ping source 172.17.0.1 162.0.1.1    
PING 162.0.1.1 (162.0.1.1): 56 data bytes
64 bytes from 162.0.1.1: icmp_seq=0 ttl=64 time=17.375 ms
64 bytes from 162.0.1.1: icmp_seq=1 ttl=64 time=29.615 ms
64 bytes from 162.0.1.1: icmp_seq=2 ttl=64 time=57.302 ms
^C
--- 162.0.1.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 17.375/34.764/57.302/16.702 ms

root@R1> ping source 172.17.0.1 162.0.1.2    
PING 162.0.1.2 (162.0.1.2): 56 data bytes
64 bytes from 162.0.1.2: icmp_seq=0 ttl=64 time=72.852 ms
64 bytes from 162.0.1.2: icmp_seq=1 ttl=64 time=291.280 ms
64 bytes from 162.0.1.2: icmp_seq=2 ttl=64 time=68.825 ms
64 bytes from 162.0.1.2: icmp_seq=3 ttl=64 time=9.527 ms
^C
--- 162.0.1.2 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 9.527/110.621/291.280/107.274 ms
---------
root@R2> ping 162.0.1.1 source 172.17.0.2 
PING 162.0.1.1 (162.0.1.1): 56 data bytes
64 bytes from 162.0.1.1: icmp_seq=0 ttl=63 time=128.354 ms
64 bytes from 162.0.1.1: icmp_seq=1 ttl=63 time=139.167 ms
64 bytes from 162.0.1.1: icmp_seq=2 ttl=63 time=10.518 ms
^C
--- 162.0.1.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 10.518/92.680/139.167/58.265 ms

root@R2> ping 162.0.1.2 source 172.17.0.2    
PING 162.0.1.2 (162.0.1.2): 56 data bytes
64 bytes from 162.0.1.2: icmp_seq=0 ttl=63 time=93.812 ms
64 bytes from 162.0.1.2: icmp_seq=1 ttl=63 time=455.516 ms
64 bytes from 162.0.1.2: icmp_seq=2 ttl=63 time=10.314 ms
64 bytes from 162.0.1.2: icmp_seq=3 ttl=63 time=18.251 ms
^C
--- 162.0.1.2 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 10.314/144.473/455.516/182.514 ms
```
And... This is OK!!! We have much more peerings to configure, let's go. 

In R3, we have two upstream connections. With P2 we'll establish a common IPv4 session and a IPv6 session using the link-local address. But, how we can find the neighbor LLA? You'll see. 
With P3 we'll establish a IPv6 session using the IPv4 compatible address. 

First, let's establish the session with P2. The interface configuration is common, but we need to configure the family inet6 on the interface, to permit the IPv6 packets here. 
```
set interfaces ge-0/0/5 description to-P2-1
set interfaces ge-0/0/5 flexible-vlan-tagging
set interfaces ge-0/0/5 unit 301 description Upstream-BGP
set interfaces ge-0/0/5 unit 301 vlan-id 301
set interfaces ge-0/0/5 unit 301 family inet address 172.16.3.1/30
set interfaces ge-0/0/5 unit 301 family inet6
```
The policies, we'll follow our standard. We must accept only prefixes between /8 to /24, and mark these prefixes with the due communities, and we'll export our aggregate prefix of our DCs networks and Router IP for test purposes. 
```
set policy-options policy-statement Entrada-P2 term 1 from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-P2 term 1 then community add P2
set policy-options policy-statement Entrada-P2 term 1 then community add Provider
set policy-options policy-statement Entrada-P2 term 1 then accept
set policy-options policy-statement Entrada-P2 term 1v6 from route-filter 0::/0 prefix-length-range /32-/48
set policy-options policy-statement Entrada-P2 term 1v6 then community add P2
set policy-options policy-statement Entrada-P2 term 1v6 then community add Provider
set policy-options policy-statement Entrada-P2 term 1v6 then accept
set policy-options policy-statement Entrada-P2 then reject

set policy-options policy-statement Saida-P2 term Customers from community Customer
set policy-options policy-statement Saida-P2 term Customers then accept
set policy-options policy-statement Saida-P2 term DCs from route-filter 172.17.0.0/22 exact
set policy-options policy-statement Saida-P2 term DCs then accept
set policy-options policy-statement Saida-P2 then reject
```
Now, we need to configure the BGP session, but we have a problem here. We need to know what is the LLA of our neighbor. (Oh, but... how the neighbor knows our LLA? Because I made this to practice, it`s not obviously?!?)

To find this, We can use the "monitor traffic" tool of Junos. 
```
root@R3> monitor traffic interface ge-0/0/5.301 no-resolve detail 
Address resolution is OFF.
Listening on ge-0/0/5.301, capture size 1578 bytes
...
14:05:05.416080  In IP6 (class 0xc0, flowlabel 0x06343, hlim 1, next-header: TCP (6), length: 91) fe80::205:8601:2d71:9500.60541 > fe80::52e4:d701:2d00:a07.179: P 1:60(59) ack 1 win 17136 <nop,nop,timestamp 66792071 2815383768>: BGP, length: 59
        Open Message (1), length: 59
          Version 4, my AS 65502, Holdtime 90s, ID 172.16.3.2
          Optional parameters, length: 30
            Option Capabilities Advertisement (2), length: 6
              Multiprotocol Extensions (1), length: 4
                AFI IPv6 (2), SAFI Unicast (1)
            Option Capabilities Advertisement (2), length: 2
              Route Refresh (Cisco) (128), length: 0
            Option Capabilities Advertisement (2), length: 2
              Route Refresh (2), length: 0
            Option Capabilities Advertisement (2), length: 4
              Graceful Restart (64), length: 2
                Restart Flags: [none], Restart Time 120s
            Option Capabilities Advertisement (2), length: 6
              32-Bit AS Number (65), length: 4
                 4 Byte AS 65502
...
```
In our interfaces we are receiving the packets from fe80::205:8601:2d71:9500. So, let's made the configuration now:
```
set protocols bgp group eBGP-AS65502-Provider2 type external
set protocols bgp group eBGP-AS65502-Provider2 description eBGP-AS65502-Provider2
set protocols bgp group eBGP-AS65502-Provider2 log-updown
set protocols bgp group eBGP-AS65502-Provider2 import Entrada-P2
set protocols bgp group eBGP-AS65502-Provider2 export Saida-P2
set protocols bgp group eBGP-AS65502-Provider2 peer-as 65502
set protocols bgp group eBGP-AS65502-Provider2 neighbor 172.16.3.2 family inet unicast
set protocols bgp group eBGP-AS65502-Provider2 neighbor fe80::205:8601:2d71:9500 local-interface ge-0/0/5.301
set protocols bgp group eBGP-AS65502-Provider2 neighbor fe80::205:8601:2d71:9500 family inet6 unicast
set protocols bgp group eBGP-AS65502-Provider2 neighbor fe80::205:8601:2d71:9500 export Saida-P2
```
And let's check this:
```
root@R3> show bgp summary group eBGP-AS65502-Provider2    
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 7 Peers: 7 Down peers: 3
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.rtarget.0        
                       1          0          0          0          0          0
inet.0               
                     154        148          0          0          0          0
inet6.0              
                      26          6          0          0          0          0
inet.3               
                       8          1          0          0          0          0
bgp.l3vpn.0          
                      57         57          0          0          0          0
bgp.l3vpn-inet6.0    
                       2          2          0          0          0          0
bgp.l2vpn.0          
                      11         11          0          0          0          0
bgp.evpn.0           
                      29         29          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.3.2            65502          7          8       0       1           7 Establ
  inet.0: 31/33/33/0
fe80::205:8601:2d71:9500%ge-0/0/5.301       65502          7          3       0       1           3 Establ
  inet6.0: 1/3/3/0
```
Ok, both sessions are established and we are receiving routes, let's check which:
```
root@R3> show route receive-protocol bgp 172.16.3.2 table inet.0                  

inet.0: 226 destinations, 233 routes (221 active, 0 holddown, 5 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  1.1.1.0/24              172.16.3.2                              65502 111 I
  8.8.8.0/24              172.16.3.2                              65502 888 I
* 200.2.0.0/24            172.16.3.2                              65502 I
* 200.2.1.0/24            172.16.3.2                              65502 I
* 200.2.2.0/24            172.16.3.2                              65502 I
* 200.2.3.0/24            172.16.3.2                              65502 I
* 200.2.4.0/24            172.16.3.2                              65502 I
* 200.2.5.0/24            172.16.3.2                              65502 I
* 200.2.6.0/24            172.16.3.2                              65502 I
* 200.2.7.0/24            172.16.3.2                              65502 I
* 200.2.8.0/24            172.16.3.2                              65502 I
* 200.2.9.0/24            172.16.3.2                              65502 I
* 200.2.10.0/24           172.16.3.2                              65502 I
* 200.2.11.0/24           172.16.3.2                              65502 I
* 200.2.12.0/24           172.16.3.2                              65502 I
* 200.2.13.0/24           172.16.3.2                              65502 I
* 200.2.14.0/24           172.16.3.2                              65502 I
* 200.2.15.0/24           172.16.3.2                              65502 I
* 200.2.16.0/24           172.16.3.2                              65502 I
* 200.2.17.0/24           172.16.3.2                              65502 I
* 200.2.18.0/24           172.16.3.2                              65502 I
* 200.2.19.0/24           172.16.3.2                              65502 I
* 200.2.20.0/24           172.16.3.2                              65502 I
* 200.2.21.0/24           172.16.3.2                              65502 I
* 200.2.22.0/24           172.16.3.2                              65502 I
* 200.2.23.0/24           172.16.3.2                              65502 I
* 200.2.24.0/24           172.16.3.2                              65502 I
* 200.2.25.0/24           172.16.3.2                              65502 I
* 200.2.26.0/24           172.16.3.2                              65502 I
* 200.2.27.0/24           172.16.3.2                              65502 I
* 200.2.28.0/24           172.16.3.2                              65502 I
* 200.2.29.0/24           172.16.3.2                              65502 I
* 200.2.30.0/24           172.16.3.2                              65502 I
root@R3> show route receive-protocol bgp fe80::205:8601:2d71:9500%ge-0/0/5.301 table inet6.0 

inet6.0: 34 destinations, 55 routes (34 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  2804:db8:111::/48       fe80::205:8601:2d71:9500                65502 111 I
  2804:db8:888::/48       fe80::205:8601:2d71:9500                65502 888 I
* 2804:2002::/48          fe80::205:8601:2d71:9500                65502 I
```
And, everything looks ok! But, let's test this. For our IPv6 peerings, let's export the fd10:faca:f0fa::/48, this way, we can test the connectivity with our loopbacks. So, we need to add one more term in our export policy:
```
set routing-options rib inet6.0 aggregate route fd10:faca:f0fa::/48 as-path aggregator 65020 10.0.0.3
set policy-options policy-statement Saida-P2 term v6 from route-filter fd10:faca:f0fa::/48 exact
set policy-options policy-statement Saida-P2 term v6 then accept
```
We can do this on all routers, similarly. 

Now, it's time to check the connectivity between the routers:
```
root@R3> ping 200.2.0.1 source 172.17.0.3 
PING 200.2.0.1 (200.2.0.1): 56 data bytes
64 bytes from 200.2.0.1: icmp_seq=0 ttl=64 time=10.829 ms
64 bytes from 200.2.0.1: icmp_seq=1 ttl=64 time=4.666 ms
64 bytes from 200.2.0.1: icmp_seq=2 ttl=64 time=93.363 ms
^C
--- 200.2.0.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 4.666/36.286/93.363/40.438 ms
root@R3> ping source fd10:faca:f0fa::3 2804:2002:: 
PING6(56=40+8+8 bytes) fd10:faca:f0fa::3 --> 2804:2002::
16 bytes from 2804:2002::, icmp_seq=0 hlim=64 time=270.940 ms
16 bytes from 2804:2002::, icmp_seq=1 hlim=64 time=45.279 ms
16 bytes from 2804:2002::, icmp_seq=2 hlim=64 time=7.579 ms
^C
--- 2804:2002:: ping6 statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 7.579/107.933/270.940/116.287 ms
```
And, this is a success! We can jump into the configuration with the P3. 

In this scenario, we'll use the IPv6 IPv4 compatible, or most knowly, the IPv4-mapped IPv6 address. You know what is this? A technique used to map 32-bit IPv4 addresses into the 128-bit IPv6 space, allowing IPv6-only applications to communicate with IPv4 nodes. The format is ::ffff:w.x.y.z, where w.x.y.z is the IPv4 address, we'll explore this now. 

This time, in our BGP session we'll speak the family inet and the family inet6 using the BGP session in IPv6 only. And the IPv4-mapped IPv6 address will work as recursive next-hop, because the routes received will go with the next-hop ::ffff:w.x.y.z, look here:
```
set interfaces ge-0/0/6 description to-P3-1
set interfaces ge-0/0/6 flexible-vlan-tagging
set interfaces ge-0/0/6 unit 302 description Upstream-BGP
set interfaces ge-0/0/6 unit 302 vlan-id 302
set interfaces ge-0/0/6 unit 302 family inet address 172.16.3.5/30
set interfaces ge-0/0/6 unit 302 family inet6 address ::ffff:172.16.3.5/126
```
The policies are similary with P2 policies:
```
set policy-options policy-statement Entrada-P3 term 1 from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-P3 term 1 then community add P3
set policy-options policy-statement Entrada-P3 term 1 then community add Provider
set policy-options policy-statement Entrada-P3 term 1 then accept
set policy-options policy-statement Entrada-P3 term 1v6 from route-filter 0::/0 prefix-length-range /32-/48
set policy-options policy-statement Entrada-P3 term 1v6 then community add P3
set policy-options policy-statement Entrada-P3 term 1v6 then community add Provider
set policy-options policy-statement Entrada-P3 term 1v6 then accept
set policy-options policy-statement Entrada-P3 then reject

set policy-options policy-statement Saida-P3 term Customers from community Customer
set policy-options policy-statement Saida-P3 term Customers then accept
set policy-options policy-statement Saida-P3 term DCs from route-filter 172.17.0.0/22 exact
set policy-options policy-statement Saida-P3 term DCs then accept
set policy-options policy-statement Saida-P3 term v6 from route-filter fd10:faca:f0fa::/48 exact
set policy-options policy-statement Saida-P3 term v6 then accept
set policy-options policy-statement Saida-P3 then reject
```
The configuration of the BGP:
```
set protocols bgp group eBGP-AS65503-Provider3 type external
set protocols bgp group eBGP-AS65503-Provider3 description eBGP-AS65503-Provider3
set protocols bgp group eBGP-AS65503-Provider3 log-updown
set protocols bgp group eBGP-AS65503-Provider3 import Entrada-P3
set protocols bgp group eBGP-AS65503-Provider3 family inet unicast
set protocols bgp group eBGP-AS65503-Provider3 family inet6 unicast
set protocols bgp group eBGP-AS65503-Provider3 export Saida-P3
set protocols bgp group eBGP-AS65503-Provider3 peer-as 65503
set protocols bgp group eBGP-AS65503-Provider3 neighbor 172.16.3.6
```
This way, in the session with the 172.16.3.6, IPv6 routes also IPv6 routes are changed. Let's look the results:
```
root@R3> show route receive-protocol bgp 172.16.3.6 

inet.0: 226 destinations, 233 routes (221 active, 0 holddown, 5 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 1.1.1.0/24              172.16.3.6                              65503 111 I
* 8.8.8.0/24              172.16.3.6                              65503 888 I
* 10.0.1.3/32             172.16.3.6                              65503 I
* 200.3.0.0/24            172.16.3.6                              65503 I
* 200.3.1.0/24            172.16.3.6                              65503 I
* 200.3.2.0/24            172.16.3.6                              65503 I
* 200.3.3.0/24            172.16.3.6                              65503 I
* 200.3.4.0/24            172.16.3.6                              65503 I
* 200.3.5.0/24            172.16.3.6                              65503 I
* 200.3.6.0/24            172.16.3.6                              65503 I
* 200.3.7.0/24            172.16.3.6                              65503 I
* 200.3.8.0/24            172.16.3.6                              65503 I
* 200.3.9.0/24            172.16.3.6                              65503 I
* 200.3.10.0/24           172.16.3.6                              65503 I
* 200.3.11.0/24           172.16.3.6                              65503 I
* 200.3.12.0/24           172.16.3.6                              65503 I
* 200.3.13.0/24           172.16.3.6                              65503 I
* 200.3.14.0/24           172.16.3.6                              65503 I
* 200.3.15.0/24           172.16.3.6                              65503 I
* 200.3.16.0/24           172.16.3.6                              65503 I
* 200.3.17.0/24           172.16.3.6                              65503 I
* 200.3.18.0/24           172.16.3.6                              65503 I
* 200.3.19.0/24           172.16.3.6                              65503 I
* 200.3.20.0/24           172.16.3.6                              65503 I
* 200.3.21.0/24           172.16.3.6                              65503 I
* 200.3.22.0/24           172.16.3.6                              65503 I
* 200.3.23.0/24           172.16.3.6                              65503 I
* 200.3.24.0/24           172.16.3.6                              65503 I
* 200.3.25.0/24           172.16.3.6                              65503 I
* 200.3.26.0/24           172.16.3.6                              65503 I
* 200.3.27.0/24           172.16.3.6                              65503 I
* 200.3.28.0/24           172.16.3.6                              65503 I
* 200.3.29.0/24           172.16.3.6                              65503 I
* 200.3.30.0/24           172.16.3.6                              65503 I

inet6.0: 35 destinations, 56 routes (35 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 2804:db8:111::/48       ::ffff:172.16.3.6                       65503 111 I
* 2804:db8:888::/48       ::ffff:172.16.3.6                       65503 888 I
* 2804:2003::/48          ::ffff:172.16.3.6                       65503 I
```
Remember the IPv4-mapped IPv6 address, the existence of that configuration active these routes. 

Ok, everything looks good, but to confirm this, we need to check the connectivity:
```
root@R3> ping 2804:2003:: source fd10:faca:f0fa::3 
PING6(56=40+8+8 bytes) fd10:faca:f0fa::3 --> 2804:2003::
16 bytes from 2804:2003::, icmp_seq=0 hlim=64 time=30.633 ms
16 bytes from 2804:2003::, icmp_seq=1 hlim=64 time=231.248 ms
16 bytes from 2804:2003::, icmp_seq=2 hlim=64 time=18.939 ms
^C
--- 2804:2003:: ping6 statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 18.939/93.607/231.248/97.444 ms

root@R3> ping 200.3.0.1 source 172.17.0.3 
PING 200.3.0.1 (200.3.0.1): 56 data bytes
64 bytes from 200.3.0.1: icmp_seq=0 ttl=64 time=45.296 ms
64 bytes from 200.3.0.1: icmp_seq=1 ttl=64 time=7.226 ms
64 bytes from 200.3.0.1: icmp_seq=2 ttl=64 time=5.994 ms
^C
--- 200.3.0.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 5.994/19.505/45.296/18.244 ms
```
And... everything is working perfectly!!! 

Now, let's configure our last Provider peering. 
In R7 and R8 we have a connection with the Provider 1, or, simply P1. This is a common dual-stack connection. But, we need to learn something, if this won't in the kind of the session, will be in the policies haha. 

In this case, in the import, we need to install ONLY routes of the AS65501, and reject the remaining. We also must prefer the upload via R8. 
And, in the export, we must send the prefixes of our AS with the no-export well-known community (the customers prefixes will be exported normally), this way the P1 will not propagate our prefixes to the internet. 

So, with the constraints defined, let's do that:

First, we need to create an as-path do define a "matcher". We will use a regex to match only the routes originated by the AS65501
```
set policy-options as-path AS65501 .*65501
```
Ok, now we need to define the no-export community to add this in our prefixes:
```
set policy-options community no-export members no-export
```
With this, we can define our policies, let's go:

R7:
```
set policy-options policy-statement Entrada-P1 term 1 from as-path AS65501
set policy-options policy-statement Entrada-P1 term 1 from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-P1 term 1 then community add P1
set policy-options policy-statement Entrada-P1 term 1 then community add Provider
set policy-options policy-statement Entrada-P1 term 1 then accept
set policy-options policy-statement Entrada-P1 term 1v6 from as-path AS65501
set policy-options policy-statement Entrada-P1 term 1v6 from route-filter 0::/0 prefix-length-range /32-/48
set policy-options policy-statement Entrada-P1 term 1v6 then community add P1
set policy-options policy-statement Entrada-P1 term 1v6 then community add Provider
set policy-options policy-statement Entrada-P1 term 1v6 then accept
set policy-options policy-statement Entrada-P1 then reject

set policy-options policy-statement Saida-P1 term Customers from community Customer
set policy-options policy-statement Saida-P1 term Customers then accept
set policy-options policy-statement Saida-P1 term DCs from protocol aggregate
set policy-options policy-statement Saida-P1 term DCs from route-filter 172.17.0.0/22 exact
set policy-options policy-statement Saida-P1 term DCs then community add no-export
set policy-options policy-statement Saida-P1 term DCs then accept
set policy-options policy-statement Saida-P1 term v6 from route-filter fd10:faca:f0fa::/48 exact
set policy-options policy-statement Saida-P1 term v6 then community add no-export
set policy-options policy-statement Saida-P1 term v6 then accept
set policy-options policy-statement Saida-P1 then reject
```
R8: You can note here, to direct the upload for the R8 session, we can use local-preference, I am setting the LP 
```
set policy-options policy-statement Entrada-P1 term 1 from as-path AS65501
set policy-options policy-statement Entrada-P1 term 1 from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-P1 term 1 then local-preference 200
set policy-options policy-statement Entrada-P1 term 1 then community add P1
set policy-options policy-statement Entrada-P1 term 1 then community add Provider
set policy-options policy-statement Entrada-P1 term 1 then accept
set policy-options policy-statement Entrada-P1 term 1v6 from as-path AS65501
set policy-options policy-statement Entrada-P1 term 1v6 from route-filter 0::/0 prefix-length-range /32-/48
set policy-options policy-statement Entrada-P1 term 1v6 then local-preference 200
set policy-options policy-statement Entrada-P1 term 1v6 then community add P1
set policy-options policy-statement Entrada-P1 term 1v6 then community add Provider
set policy-options policy-statement Entrada-P1 term 1v6 then accept
set policy-options policy-statement Entrada-P1 then reject

set policy-options policy-statement Saida-C5 term Default from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Saida-C5 term Default then accept
set policy-options policy-statement Saida-C5 term General from community Customer
set policy-options policy-statement Saida-C5 term General from community Provider
set policy-options policy-statement Saida-C5 term General from community IX
set policy-options policy-statement Saida-C5 term General then accept
set policy-options policy-statement Saida-C5 term DCs from route-filter 172.17.0.0/22 exact
set policy-options policy-statement Saida-C5 term DCs then accept
set policy-options policy-statement Saida-C5 term v6 from protocol aggregate
set policy-options policy-statement Saida-C5 term v6 from route-filter fd10:faca:f0fa::/48 exact
set policy-options policy-statement Saida-C5 term v6 then accept
set policy-options policy-statement Saida-C5 then reject
```
And, let's to the BGP configuration:
R7:
```
set protocols bgp group eBGP-AS65501 type external
set protocols bgp group eBGP-AS65501 description eBGP-AS65501
set protocols bgp group eBGP-AS65501 log-updown
set protocols bgp group eBGP-AS65501 import Entrada-P1
set protocols bgp group eBGP-AS65501 export Saida-P1
set protocols bgp group eBGP-AS65501 peer-as 65501
set protocols bgp group eBGP-AS65501 neighbor 172.16.7.6 family inet unicast
set protocols bgp group eBGP-AS65501 neighbor fc09:c0:ffee:7::2 family inet6 unicast
```
R8:
```
set protocols bgp group eBGP-AS65501 type external
set protocols bgp group eBGP-AS65501 description eBGP-AS65501
set protocols bgp group eBGP-AS65501 log-updown
set protocols bgp group eBGP-AS65501 import Entrada-P1
set protocols bgp group eBGP-AS65501 export Saida-P1
set protocols bgp group eBGP-AS65501 peer-as 65501
set protocols bgp group eBGP-AS65501 neighbor 172.16.8.2 family inet unicast
set protocols bgp group eBGP-AS65501 neighbor fc09:c0:ffee:8::2 family inet6 unicast
```

Let's check the results:
```
root@R7> show bgp summary group eBGP-AS65501                            
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 3 Peers: 4 Down peers: 1
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.rtarget.0        
                       0          0          0          0          0          0
inet.0               
                      53         31         20         20         40          0
inet6.0              
                       3          1          0          0          0          0
bgp.mvpn.0           
                       0          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.7.6            65501      27342      27024       0       0 1w1d 13:51:38 Establ
  inet.0: 31/33/31/0
fc09:c0:ffee:7::2       65501      27342      27021       0       0 1w1d 13:51:32 Establ
  inet6.0: 1/3/1/0

root@R8> show bgp summary group eBGP-AS65501 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 3 Peers: 4 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.rtarget.0        
                       1          1          0          0          0          0
inet.0               
                     156        153          0          0          0          0
inet6.0              
                      26          6          0          0          0          0
bgp.l3vpn.0          
                      15         15          0          0          0          0
bgp.l3vpn-inet6.0    
                       2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.8.2            65501      27326      27179       0       0 1w1d 13:46:53 Establ
  inet.0: 31/33/31/0
fc09:c0:ffee:8::2       65501      27325      27157       0       0 1w1d 13:46:45 Establ
  inet6.0: 1/3/1/0
```
Here you can see that the routes are active in both routers, this happens because we don't have configured our iBGP yet. In the next chapter of our journey, we'll configure this iBGP and the routes will become inactive. 

But, let's desconsiderate this situation, the best thing to do is check if we are setting the local-pref value to 200 in R8:
```
root@R8> show route protocol bgp aspath-regex .*65501                 

inet.0: 224 destinations, 228 routes (222 active, 0 holddown, 4 hidden)
+ = Active Route, - = Last Active, * = Both

200.1.0.0/24       *[BGP/170] 1w1d 13:52:39, localpref 200
                      AS path: 65501 I, validation-state: unverified
                    >  to 172.16.8.2 via ge-0/0/1.801
200.1.1.0/24       *[BGP/170] 1w1d 13:52:39, localpref 200
                      AS path: 65501 I, validation-state: unverified
                    >  to 172.16.8.2 via ge-0/0/1.801
200.1.2.0/24       *[BGP/170] 1w1d 13:52:39, localpref 200
                      AS path: 65501 I, validation-state: unverified
                    >  to 172.16.8.2 via ge-0/0/1.801
.........
inet6.0: 34 destinations, 55 routes (34 active, 0 holddown, 2 hidden)
+ = Active Route, - = Last Active, * = Both

2804:2001::/48     *[BGP/170] 1w1d 13:52:32, localpref 200
                      AS path: 65501 I, validation-state: unverified
                    >  to fc09:c0:ffee:8::2 via ge-0/0/1.801
```
And, this is ok!!! Let's check the communication to finish this. 
```
root@R7> ping 200.1.0.1 source 172.17.0.7 
PING 200.1.0.1 (200.1.0.1): 56 data bytes
64 bytes from 200.1.0.1: icmp_seq=0 ttl=63 time=137.147 ms
64 bytes from 200.1.0.1: icmp_seq=1 ttl=63 time=39.417 ms
^C
--- 200.1.0.1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 39.417/88.282/137.147/48.865 ms

root@R7> ping 200.1.0.2 source 172.17.0.7    
PING 200.1.0.2 (200.1.0.2): 56 data bytes
64 bytes from 200.1.0.2: icmp_seq=0 ttl=64 time=8.113 ms
64 bytes from 200.1.0.2: icmp_seq=1 ttl=64 time=9.562 ms
^C
--- 200.1.0.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 8.113/8.837/9.562/0.725 ms

root@R7> ping 2804:2001::1 source fd10:faca:f0fa::7  
PING6(56=40+8+8 bytes) fd10:faca:f0fa::7 --> 2804:2001::1
16 bytes from 2804:2001::1, icmp_seq=0 hlim=63 time=40.516 ms
16 bytes from 2804:2001::1, icmp_seq=1 hlim=63 time=17.326 ms
^C
--- 2804:2001::1 ping6 statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 17.326/28.921/40.516/11.595 ms

root@R7> ping 2804:2001::2 source fd10:faca:f0fa::7    
PING6(56=40+8+8 bytes) fd10:faca:f0fa::7 --> 2804:2001::2
16 bytes from 2804:2001::2, icmp_seq=0 hlim=64 time=14.126 ms
16 bytes from 2804:2001::2, icmp_seq=1 hlim=64 time=10.373 ms
^C
--- 2804:2001::2 ping6 statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 10.373/12.249/14.126/1.877 ms

root@R8> ping 200.1.0.1 source 172.17.0.8  
PING 200.1.0.1 (200.1.0.1): 56 data bytes
64 bytes from 200.1.0.1: icmp_seq=0 ttl=64 time=19.371 ms
64 bytes from 200.1.0.1: icmp_seq=1 ttl=64 time=188.420 ms
^C
--- 200.1.0.1 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 19.371/103.895/188.420/84.524 ms

root@R8> ping 200.1.0.2 source 172.17.0.8    
PING 200.1.0.2 (200.1.0.2): 56 data bytes
64 bytes from 200.1.0.2: icmp_seq=0 ttl=63 time=37.143 ms
64 bytes from 200.1.0.2: icmp_seq=1 ttl=63 time=46.719 ms
^C
--- 200.1.0.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 37.143/41.931/46.719/4.788 ms

root@R8> ping 2804:2001::1 source fd10:faca:f0fa::8  
PING6(56=40+8+8 bytes) fd10:faca:f0fa::8 --> 2804:2001::1
16 bytes from 2804:2001::1, icmp_seq=0 hlim=64 time=88.181 ms
16 bytes from 2804:2001::1, icmp_seq=1 hlim=64 time=14.392 ms
^C
--- 2804:2001::1 ping6 statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 14.392/51.286/88.181/36.895 ms

root@R8> ping 2804:2001::2 source fd10:faca:f0fa::8    
PING6(56=40+8+8 bytes) fd10:faca:f0fa::8 --> 2804:2001::2
16 bytes from 2804:2001::2, icmp_seq=0 hlim=63 time=22.210 ms
16 bytes from 2804:2001::2, icmp_seq=1 hlim=63 time=13.382 ms
^C
--- 2804:2001::2 ping6 statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 13.382/17.796/22.210/4.414 ms
```
And... everything is ok!!!

Let's go to the customers peering configuration! 

In R5, we have the C5 connected in two different sites, and this customer have a backdoor link between the CEs. The Customer ask us to load-balancing the traffic between the links. 
Another good practice that I've adopted is, not accept the default route from customers, after all, we are the upstream of them. 

So, with this, let's make the configuration. In import policy, we'll reject the default route, accept only routes from AS64505 (The AS-PATH matching is to avoid problems, what if the customer export the route 8.8.8.8 with the blackhole community? You will install this route with the discard action, you know, right? In the real scenario, we'll use RPKI to accept routes from customers of our customer, but this is not the case), with the blackhole community. We'll add another type of community, a community that change de local-preference in our network to provide to customer some options of traffic engineering. If our customer export a prefix with the community Cust-LP-90, then our router will set the local-pref value to 90. The another two terms will accept the customer prefixes according to our intern policy, the customer prefixes will have a local-pref value of 600, after all, we need to sell our service hahah. 

In export policy, we'll send our DFZ (With the routes installed from IX, Providers, Customers, and our aggregated route) and the default gateway (The default route was configured in previous chapters, do you remember that?). 
```
set policy-options policy-statement Entrada-C5 term deny-gw from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Entrada-C5 term deny-gw then reject
set policy-options policy-statement Entrada-C5 term BH from as-path AS64505
set policy-options policy-statement Entrada-C5 term BH from community Blackhole
set policy-options policy-statement Entrada-C5 term BH from route-filter 0.0.0.0/0 upto /32
set policy-options policy-statement Entrada-C5 term BH then next-hop discard
set policy-options policy-statement Entrada-C5 term BH then accept
set policy-options policy-statement Entrada-C5 term Baixa-LP from as-path AS64505
set policy-options policy-statement Entrada-C5 term Baixa-LP from community Cust-LP-90
set policy-options policy-statement Entrada-C5 term Baixa-LP from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-C5 term Baixa-LP then local-preference 90
set policy-options policy-statement Entrada-C5 term Baixa-LP then accept
set policy-options policy-statement Entrada-C5 term 1 from as-path AS64505
set policy-options policy-statement Entrada-C5 term 1 from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-C5 term 1 then local-preference 600
set policy-options policy-statement Entrada-C5 term 1 then community add C5
set policy-options policy-statement Entrada-C5 term 1 then community add Customer
set policy-options policy-statement Entrada-C5 term 1 then accept
set policy-options policy-statement Entrada-C5 term 1-v6 from as-path AS64505
set policy-options policy-statement Entrada-C5 term 1-v6 from route-filter 0::/0 prefix-length-range /32-/48
set policy-options policy-statement Entrada-C5 term 1-v6 then local-preference 600
set policy-options policy-statement Entrada-C5 term 1-v6 then community add C5
set policy-options policy-statement Entrada-C5 term 1-v6 then community add Customer
set policy-options policy-statement Entrada-C5 term 1-v6 then accept
set policy-options policy-statement Entrada-C5 then reject

set policy-options policy-statement Saida-C5 term Default from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Saida-C5 term Default then accept
set policy-options policy-statement Saida-C5 term General from community Customer
set policy-options policy-statement Saida-C5 term General from community Provider
set policy-options policy-statement Saida-C5 term General from community IX
set policy-options policy-statement Saida-C5 term General then accept
set policy-options policy-statement Saida-C5 term DCs from route-filter 172.17.0.0/22 exact
set policy-options policy-statement Saida-C5 term DCs then accept
set policy-options policy-statement Saida-C5 then reject
```
With the policies defined, we can configure the BGP sessions. To permit the premisse of load-balancing, we need to configure the multipath.
```
set protocols bgp group eBGP-AS64505 type external
set protocols bgp group eBGP-AS64505 description eBGP-AS64505
set protocols bgp group eBGP-AS64505 log-updown
set protocols bgp group eBGP-AS64505 damping
set protocols bgp group eBGP-AS64505 import Entrada-C5
set protocols bgp group eBGP-AS64505 export Saida-C5
set protocols bgp group eBGP-AS64505 peer-as 64505
set protocols bgp group eBGP-AS64505 multipath
set protocols bgp group eBGP-AS64505 neighbor 172.16.5.2 family inet unicast
set protocols bgp group eBGP-AS64505 neighbor fc09:c0:ffee:5::2 family inet6 unicast
set protocols bgp group eBGP-AS64505 neighbor 172.16.5.6 family inet unicast
set protocols bgp group eBGP-AS64505 neighbor fc09:c0:ffee:5::6 family inet6 unicast
```
But, this is sufficient to do the load-balancing? Let's check this:

```
root@R5> show route 201.5.0.0/24 active-path      

inet.0: 231 destinations, 252 routes (226 active, 0 holddown, 8 hidden)
+ = Active Route, - = Last Active, * = Both

201.5.0.0/24       *[BGP/170] 00:01:44, localpref 600, from 172.16.5.2
                      AS path: 64505 I, validation-state: unverified
                       to 172.16.5.2 via ge-0/0/8.501
                    >  to 172.16.5.6 via ge-0/0/9.502

root@R5> show route 2804:2015::/48 active-path    

inet6.0: 41 destinations, 66 routes (38 active, 0 holddown, 8 hidden)
+ = Active Route, - = Last Active, * = Both

2804:2015::/48     *[BGP/170] 00:01:47, localpref 600, from fc09:c0:ffee:5::2
                      AS path: 64505 I, validation-state: unverified
                       to fc09:c0:ffee:5::2 via ge-0/0/8.501
                    >  to fc09:c0:ffee:5::6 via ge-0/0/9.502
```
We are receiving the routes from both CEs, but to check if the load-balancing are happen, we need to check our FIB:
```
root@R5> show route forwarding-table destination 201.5.0.0/24    
Routing table: default.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
201.5.0.0/24       user     0 172.16.5.6         ucst      702    14 ge-0/0/9.502
root@R5> show route forwarding-table destination 2804:2015::/48             
Routing table: default.inet6
Internet6:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
2804:2015::/48     user     0 fc09:c0:ffee:5::6  ucst      795     4 ge-0/0/9.502
```
And, we can see that only one interface are used to forward the traffic for these prefixes. 

To do the load-balance, we need to make a policy and apply this in the forwarding-table. 
```            
set policy-options policy-statement load-balance then load-balance per-flow
set routing-options forwarding-table export load-balance
```
With this, all the traffic will be load-balance in per-flow mode. Now, we can check the results: 
```
root@R5> show route forwarding-table destination 201.5.0.0/24 
Routing table: default.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
201.5.0.0/24       user     0                    ulst  1048583    13
                              172.16.5.2         ucst      698     4 ge-0/0/8.501
                              172.16.5.6         ucst      702     4 ge-0/0/9.502

root@R5> show route forwarding-table destination 2804:2015::/48 
Routing table: default.inet6
Internet6:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
2804:2015::/48     user     0                    ulst  1048578     2
                              fc09:c0:ffee:5::2  ucst      705     4 ge-0/0/8.501
                              fc09:c0:ffee:5::6  ucst      795     4 ge-0/0/9.502
```
And, now it's ok!!! We are load-balancing the traffic correctly, but we need to check the connectiviy to say that is 100%. 

