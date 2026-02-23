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
| R5 | ge-0/0/9.502 | 172.16.5.5/30 | N/A | 502 | Customer 5 | 64505 |
| R6 | ge-0/0/8.601 | 172.16.6.1/30 | N/A | 601 | Customer 1 | 64501 |
| R6 | ge-0/0/5.602 | 172.16.6.5/30 | N/A | 602 | Customer 2 | 64502 |
| R6 | ge-0/0/9.603 | 172.16.6.9/30 | N/A | 603 | Customer 2 | 64502 |
| R7 | ge-0/0/5.701 | 172.16.7.1/30 | N/A | 701 | Provider 1 | 65501 |
| R7 | ge-0/0/1.702 | 172.16.7.5/30 | N/A | 702 | Customer 1 | 64501 |
| R8 | ge-0/0/1.801 | 172.16.8.1/30 | N/A | 801 | Provider 1 | 65501 |

For all sessions, we`ll need to log the state changes of the session in the logs. 
So we need to apply the _log-updown_ in every session that we configure in our network. 

And, in every session we need to receive only the routes bigger than /24 and less than /8, otherwise, we`ll reject the routes. 

Now, I'll follow the peering cases of every router. 
In the IX environment, we have two routers, and we need to establish the session with both of them, in R1 and R2: 
The IX-1 have the IP 192.168.12.253 and the IX-2 192.168.12.254. Here I have a tip to give to you, always mark a community in the import policy, this way, we can handle the routes imported by this peering easily. 
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
set interfaces ge-0/0/5 unit 120 description ATM
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


