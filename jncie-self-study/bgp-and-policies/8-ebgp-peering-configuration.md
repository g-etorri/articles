# eBGP Peering Configuration

Hello guys! Today we are going to configure eBGP peerings within our ASN. Here is our topology:
<img width="979" height="833" alt="image" src="https://github.com/user-attachments/assets/06333cff-6ecc-4727-aa40-53a288f4f13b" />

For this task, we will do some particularities to practice our BGP handling. 

First, we need to gather the information required to configure the interfaces, IP addressing, and BGP sessions. Please refer to the table below:
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

For all BGP sessions, we must ensure that state changes are recorded in the system logs. Therefore, we need to apply the log-updown command to every neighbor within our network.

Additionally, we must implement a prefix filter for all sessions. We will only accept routes with a mask length greater than /8 and less than or equal to /24. Any routes falling outside this range will be rejected.

Let's start with the IX environment. We have two routers (R1 and R2) that need to establish sessions with two external peers:

IX-1: 192.168.12.254

IX-2: 192.168.12.252

Pro Tip: Always tag imported routes with a specific BGP Community in your import policy. This best practice makes it much easier to manage and manipulate these routes later in your control plane.

First, let’s configure the interfaces: Since the interface configuration follows a similar pattern across all routers in this lab, I will show the setup for R1 and R2, and omit the remaining interfaces for brevity.

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
Next, we define the policies for these sessions. As mentioned, we’ll start by defining our BGP Communities. We will implement these across every router in our network to maintain consistency.

Let’s create generic communities for role identification, including the Blackhole community and specific peering roles:
```
set policy-options community Customer members 65020:51
set policy-options community IX members 65020:1620
set policy-options community Provider members 65020:50
set policy-options community Blackhole members 6450.:666
```
Note on Blackhole Community: By using a Regular Expression (regex), we can identify blackhole requests from various customer ASNs (e.g., 64501:666, 64502:666). This allows our routers to dynamically identify and discard traffic based on the customer’s specific AS.

We can also configure granular communities for each neighbor to allow for more flexible routing policies:
```
set policy-options community C1 members 65020:64501
set policy-options community C2 members 65020:64502
set policy-options community C5 members 65020:64505
set policy-options community P1 members 65020:65501
set policy-options community P2 members 65020:65502
set policy-options community P3 members 65020:65503
```

For our IX peerings, we will accept all routes within the /8 to /24 range and tag them with the IX community.
```
set policy-options policy-statement Entrada-IX term 1 from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-IX term 1 then community add IX
set policy-options policy-statement Entrada-IX term 1 then accept
```
Now, we need to define what we will advertise to the IX. We want to export our Customer routes and our Data Center (DC) prefixes.

As discussed in previous articles, our DC prefixes are 172.17.1.0/24, 172.17.2.0/24, and 172.17.3.0/24. To optimize our routing table, we will summarize these into a single /22 aggregate route.
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

Finally, let’s apply the configuration to the BGP protocol. In this scenario, we want to influence inbound traffic to prefer R1 for downloads. To achieve this, we will use the MED attribute, applying a metric (10) on R2 to make it less preferred.

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
set protocols bgp group eBGP-IX-LAB type external
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

Now, let's verify what we are receiving from the IX. Running the command on R2, we can see the routes being learned:
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
Everything looks correct! We are receiving prefixes from famous DNS providers (1.1.1.1 and 8.8.8.8) along with other IX prefixes.

Next, let's confirm that R2 is correctly exporting our aggregated DC prefix with a MED of 10:
```
root@R2> show route advertising-protocol bgp 192.168.12.253 172.17.0.0/22 

inet.0: 225 destinations, 277 routes (220 active, 0 holddown, 5 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 172.17.0.0/22           Self                 10                 I
```

To perform connectivity tests, I'll assign addresses from the 172.17.0.0/24 range (which is part of our advertised aggregate) to the loopback interfaces of our routers. For instance, R1 will use .1 and R2 will use .2.

Note: We use the primary keyword on the main loopback address to ensure management traffic still uses our infrastructure IP.
```
set interfaces lo0 unit 0 family inet address 10.0.0.1/32 primary
set interfaces lo0 unit 0 family inet address 172.17.0.1/32
```
Now, we can verify end-to-end connectivity by pinging the IX peers using our new loopback address as the source:
```
root@R1> ping source 172.17.0.1 162.0.1.1    
PING 162.0.1.1 (162.0.1.1): 56 data bytes
64 bytes from 162.0.1.1: icmp_seq=0 ttl=64 time=17.375 ms
64 bytes from 162.0.1.1: icmp_seq=1 ttl=64 time=29.615 ms
64 bytes from 162.0.1.1: icmp_seq=2 ttl=64 time=57.302 ms
--- 162.0.1.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 17.375/34.764/57.302/16.702 ms

root@R1> ping source 172.17.0.1 162.0.1.2    
PING 162.0.1.2 (162.0.1.2): 56 data bytes
64 bytes from 162.0.1.2: icmp_seq=0 ttl=64 time=72.852 ms
64 bytes from 162.0.1.2: icmp_seq=1 ttl=64 time=291.280 ms
64 bytes from 162.0.1.2: icmp_seq=2 ttl=64 time=68.825 ms
64 bytes from 162.0.1.2: icmp_seq=3 ttl=64 time=9.527 ms
--- 162.0.1.2 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 9.527/110.621/291.280/107.274 ms
---------
root@R2> ping 162.0.1.1 source 172.17.0.2 
PING 162.0.1.1 (162.0.1.1): 56 data bytes
64 bytes from 162.0.1.1: icmp_seq=0 ttl=63 time=128.354 ms
64 bytes from 162.0.1.1: icmp_seq=1 ttl=63 time=139.167 ms
64 bytes from 162.0.1.1: icmp_seq=2 ttl=63 time=10.518 ms
--- 162.0.1.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 10.518/92.680/139.167/58.265 ms

root@R2> ping 162.0.1.2 source 172.17.0.2    
PING 162.0.1.2 (162.0.1.2): 56 data bytes
64 bytes from 162.0.1.2: icmp_seq=0 ttl=63 time=93.812 ms
64 bytes from 162.0.1.2: icmp_seq=1 ttl=63 time=455.516 ms
64 bytes from 162.0.1.2: icmp_seq=2 ttl=63 time=10.314 ms
64 bytes from 162.0.1.2: icmp_seq=3 ttl=63 time=18.251 ms
--- 162.0.1.2 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 10.314/144.473/455.516/182.514 ms
```
Success! The peering is stable, policies are applied, and we have reachability. But we're just getting started—there are plenty of other peerings to configure. Let’s keep going!

In R3, we have two upstream connections. First, we will establish peering with P2 using two sessions: a standard IPv4 session and an IPv6 session using Link-Local Addresses (LLA).

The interface configuration is straightforward, but we must enable family inet6 to allow IPv6 traffic on the unit:
```
set interfaces ge-0/0/5 description to-P2-1
set interfaces ge-0/0/5 flexible-vlan-tagging
set interfaces ge-0/0/5 unit 301 description Upstream-BGP
set interfaces ge-0/0/5 unit 301 vlan-id 301
set interfaces ge-0/0/5 unit 301 family inet address 172.16.3.1/30
set interfaces ge-0/0/5 unit 301 family inet6
```
Following our standard, we will accept only prefixes within the /8 to /24 range (IPv4) and /32 to /48 (IPv6), tagging them with the appropriate communities. We will also export our aggregated DC networks and our testing loopbacks.
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
Since we are using Link-Local addresses for the IPv6 peering, we need to know the neighbor's fe80:: address. A great way to find this in Junos without documentation is using the monitor traffic tool to sniff the BGP Open messages:
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
The capture shows the neighbor's LLA as fe80::205:8601:2d71:9500. Now we can complete the BGP configuration.
Note that for LLA neighbors, you must specify the local-interface. 
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
Both sessions are now established. 

Let's verify the received routes and test connectivity using our loopbacks:
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
And everything seems fine! But let's test this. For our IPv6 peerings, we will export fd10:faca:f0fa::/48, this way we can test connectivity with our loopbacks. Therefore, we need to add one more term to our export policy:
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
--- 200.2.0.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 4.666/36.286/93.363/40.438 ms

root@R3> ping source fd10:faca:f0fa::3 2804:2002:: 
PING6(56=40+8+8 bytes) fd10:faca:f0fa::3 --> 2804:2002::
16 bytes from 2804:2002::, icmp_seq=0 hlim=64 time=270.940 ms
16 bytes from 2804:2002::, icmp_seq=1 hlim=64 time=45.279 ms
16 bytes from 2804:2002::, icmp_seq=2 hlim=64 time=7.579 ms
--- 2804:2002:: ping6 statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 7.579/107.933/270.940/116.287 ms
```
Everything is working perfectly! Now we are ready to tackle the configuration for P3

In this scenario, we will explore IPv4-mapped IPv6 addresses. This technique maps 32-bit IPv4 addresses into the 128-bit IPv6 address space, allowing IPv6-capable applications and protocols to handle IPv4 nodes seamlessly.

The format follows the pattern ::ffff:w.x.y.z, where w.x.y.z is the original IPv4 address. In our BGP session, we will exchange both family inet and family inet6 prefixes over a single session. The IPv4-mapped address will act as a recursive next-hop, meaning the received IPv6 routes will point to ::ffff:172.16.3.6.

We need to configure both the standard IPv4 address and the mapped IPv6 address on the interface:
```
set interfaces ge-0/0/6 description to-P3-1
set interfaces ge-0/0/6 flexible-vlan-tagging
set interfaces ge-0/0/6 unit 302 description Upstream-BGP
set interfaces ge-0/0/6 unit 302 vlan-id 302
set interfaces ge-0/0/6 unit 302 family inet address 172.16.3.5/30
set interfaces ge-0/0/6 unit 302 family inet6 address ::ffff:172.16.3.5/126
```
The policies for P3 follow our established baseline: filtering by prefix length and tagging with provider-specific communities.
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
Now, let's look at the BGP group configuration. Notice that we are establishing the neighbor using the IPv4 address but enabling both address families:
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
By checking the received routes, we can see the mapping in action. While IPv4 prefixes look normal, the IPv6 prefixes now use the mapped address as their next-hop:
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

To confirm that the recursive resolution is working correctly in the data plane, let's run our pings:
```
root@R3> ping 2804:2003:: source fd10:faca:f0fa::3 
PING6(56=40+8+8 bytes) fd10:faca:f0fa::3 --> 2804:2003::
16 bytes from 2804:2003::, icmp_seq=0 hlim=64 time=30.633 ms
16 bytes from 2804:2003::, icmp_seq=1 hlim=64 time=231.248 ms
16 bytes from 2804:2003::, icmp_seq=2 hlim=64 time=18.939 ms
--- 2804:2003:: ping6 statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 18.939/93.607/231.248/97.444 ms

root@R3> ping 200.3.0.1 source 172.17.0.3 
PING 200.3.0.1 (200.3.0.1): 56 data bytes
64 bytes from 200.3.0.1: icmp_seq=0 ttl=64 time=45.296 ms
64 bytes from 200.3.0.1: icmp_seq=1 ttl=64 time=7.226 ms
64 bytes from 200.3.0.1: icmp_seq=2 ttl=64 time=5.994 ms
--- 200.3.0.1 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 5.994/19.505/45.296/18.244 ms
```
Everything is working perfectly! This setup demonstrates how flexible Junos can be when handling multi-protocol BGP environments.

To conclude our Upstream configurations, we will set up the sessions with Provider 1 (P1) on R7 and R8. This is a standard dual-stack connection, but with specific traffic engineering requirements:

Strict Inbound Filtering: We must only accept routes originated by AS65501.

Inbound Traffic Engineering: We want to prefer the path via R8 for all incoming traffic from P1 (setting a higher Local Preference).

Policy Export: Our own prefixes must be advertised with the no-export well-known community, ensuring P1 does not propagate them further to the global internet. Customers' prefixes, however, will be advertised normally.

First, let's create the AS-Path regular expression to match routes originated by P1 and define the no-export community:
```
set policy-options as-path AS65501 .*65501
set policy-options community no-export members no-export
```
With this, we can define our policies, let's go:

On R8, we will apply a local-preference of 200 to make it the preferred exit point for our AS when reaching P1 destinations.
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
R8:
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
With the policies ready, we apply them to the BGP groups on both routers:
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
Here you can see that the routes are active on both routers. This is because we haven't configured iBGP yet. In the next chapter of our journey, we will configure iBGP and the routes will become inactive. 

Let's verify if R8 is correctly applying the local-preference of 200 to the routes learned from AS65501:
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
To wrap things up, we test communication from our loopbacks to the P1 network. Successful pings across both IPv4 and IPv6 confirm that our routing and policies are solid:
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
Everything is working perfectly! While routes are currently active on both routers because our iBGP mesh isn't up yet, our eBGP foundations are ready.

Now, let's configure our first customer, Customer 5 (C5), connected to R5 via two different sites. This customer has a backdoor link between their CEs and has requested load balancing across both links.

To ensure a robust and professional setup, we will implement several best practices:

Default Route Filtering: We will not accept a default route from the customer, as we are their upstream.

BGP Damping: To protect our network from instability, we’ll enable damping to penalize flapping routes.

Traffic Engineering Options: We will provide communities (e.g., Cust-LP-90) allowing the customer to lower their local preference within our AS.

Service Monetization: By default, customer routes will be assigned a local-preference of 600, ensuring they are preferred over IX or Provider routes.

The import policy handles prefix filtering, blackhole communities (with a discard next-hop), and local preference manipulation. In the export policy, we advertise the DFZ prefixes and a default gateway.
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
To allow load balancing, we must enable multipath in the BGP group. We also enable damping using default parameters.
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
Important Observation: Even with multipath enabled, Junos only installs one active path in the FIB by default. You can see this by checking the show route forwarding-table:
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

To truly achieve per-flow load balancing in the data plane, we must create a forwarding-table export policy:
```            
set policy-options policy-statement load-balance then load-balance per-flow
set routing-options forwarding-table export load-balance
```
Now, let's verify the results in the FIB. Notice the ulst (Unicast List) type, indicating multiple next-hops:
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
Final pings confirm that reachability is 100% and traffic is flowing correctly across both links: 
```
root@R5> ping 201.5.0.3 source 172.17.0.5 
PING 201.5.0.3 (201.5.0.3): 56 data bytes
64 bytes from 201.5.0.3: icmp_seq=0 ttl=64 time=5.171 ms
64 bytes from 201.5.0.3: icmp_seq=1 ttl=64 time=28.183 ms
^C
--- 201.5.0.3 ping statistics ---
3 packets transmitted, 2 packets received, 33% packet loss
round-trip min/avg/max/stddev = 5.171/16.677/28.183/11.506 ms

root@R5> ping 201.5.0.6 source 172.17.0.5    
PING 201.5.0.6 (201.5.0.6): 56 data bytes
64 bytes from 201.5.0.6: icmp_seq=0 ttl=64 time=9.743 ms
64 bytes from 201.5.0.6: icmp_seq=1 ttl=64 time=24.503 ms
64 bytes from 201.5.0.6: icmp_seq=2 ttl=64 time=33.228 ms
^C
--- 201.5.0.6 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/stddev = 9.743/22.491/33.228/9.693 ms

root@R5> ping source fd10:faca:f0fa::5 2804:2015::3    
PING6(56=40+8+8 bytes) fd10:faca:f0fa::5 --> 2804:2015::3
16 bytes from 2804:2015::3, icmp_seq=0 hlim=64 time=24.196 ms
16 bytes from 2804:2015::3, icmp_seq=1 hlim=64 time=52.234 ms
16 bytes from 2804:2015::3, icmp_seq=2 hlim=64 time=72.389 ms
^C
--- 2804:2015::3 ping6 statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 24.196/49.606/72.389/19.762 ms

root@R5> ping source fd10:faca:f0fa::5 2804:2015::6    
PING6(56=40+8+8 bytes) fd10:faca:f0fa::5 --> 2804:2015::6
16 bytes from 2804:2015::6, icmp_seq=0 hlim=64 time=6.837 ms
16 bytes from 2804:2015::6, icmp_seq=1 hlim=64 time=11.499 ms
16 bytes from 2804:2015::6, icmp_seq=2 hlim=64 time=9.209 ms
^C
--- 2804:2015::6 ping6 statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 6.837/9.182/11.499/1.903 ms
```
Success! We have successfully implemented a redundant, load-balanced customer peering with professional-grade filtering and TE options.

Now, let's configure Customer 2 (C2), who is multi-homed to R6. This scenario presents a specific challenge: the customer has two physical connections to the same CE but wants to establish the BGP session using a loopback address (172.16.6.254).

The goal is to maintain a single BGP session that remains stable as long as at least one physical link is up. Note that this customer is IPv4-only (disgusting, like github btw, ALÔ GITHUB, CADÊ O IPV6?). Additionally, they only wish to receive a default route from us.

Before the BGP session can come up, R6 must know how to reach the customer's loopback. We will use static routes pointing to both physical interfaces to enable path redundancy and load balancing: 
```
set routing-options static route 172.16.6.254/32 next-hop 172.16.6.6
set routing-options static route 172.16.6.254/32 next-hop 172.16.6.10
set policy-options policy-statement load-balance then load-balance per-flow
set routing-options forwarding-table export load-balance
```
This way, we are load-balancing the traffic to the customer:
```
root@R6> show route forwarding-table destination 172.16.6.254/32 
Routing table: default.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
172.16.6.254/32    user     1                    ulst  1048574     4
                              172.16.6.6         ucst      652     3 ge-0/0/5.602
                              172.16.6.10        ucst      653     3 ge-0/0/9.603
```
The import policy follows our security baseline (rejecting default gateways and handling blackholes), while the outbound policy is strictly limited to advertising the default route.
```
set policy-options as-path AS64502 .*64502
set policy-options policy-statement Entrada-C2 term deny-gw from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Entrada-C2 term deny-gw then reject
set policy-options policy-statement Entrada-C2 term BH from as-path AS64502
set policy-options policy-statement Entrada-C2 term BH from community Blackhole
set policy-options policy-statement Entrada-C2 term BH from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-C2 term BH then next-hop discard
set policy-options policy-statement Entrada-C2 term BH then accept
set policy-options policy-statement Entrada-C2 term Baixa-LP from as-path AS64502
set policy-options policy-statement Entrada-C2 term Baixa-LP from community Cust-LP-90
set policy-options policy-statement Entrada-C2 term Baixa-LP from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-C2 term Baixa-LP then local-preference 90
set policy-options policy-statement Entrada-C2 term Baixa-LP then accept
set policy-options policy-statement Entrada-C2 term 1 from as-path AS64502
set policy-options policy-statement Entrada-C2 term 1 from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-C2 term 1 then local-preference 600
set policy-options policy-statement Entrada-C2 term 1 then community add C2
set policy-options policy-statement Entrada-C2 term 1 then community add Customer
set policy-options policy-statement Entrada-C2 term 1 then accept
set policy-options policy-statement Entrada-C2 then reject

set policy-options policy-statement Saida-C2 term Default from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Saida-C2 term Default then accept
set policy-options policy-statement Saida-C2 then reject
```
With the policies ready, we can configure the BGP session: As you probably know, eBGP sessions have a TTL value of 1, but this isn't 100% [true](https://www.networkfuntimes.com/your-multihop-bgp-session-probably-isnt-multi-hop/). Even if our neighbors are directly connected and have the address I defined in the session, the session won't be established. Do you understand that the TTL isn't 100% true? Even if the neighbor is physically one hop away, we must use the multihop knob. This allows the session to establish and tells Junos to look into the routing table to resolve the neighbor's address.
```
set protocols bgp group eBGP-AS64502 type external
set protocols bgp group eBGP-AS64502 description eBGP-AS64502
set protocols bgp group eBGP-AS64502 multihop
set protocols bgp group eBGP-AS64502 local-address 10.0.0.6
set protocols bgp group eBGP-AS64502 log-updown
set protocols bgp group eBGP-AS64502 damping
set protocols bgp group eBGP-AS64502 import Entrada-C2
set protocols bgp group eBGP-AS64502 export Saida-C2
set protocols bgp group eBGP-AS64502 peer-as 64502
set protocols bgp group eBGP-AS64502 neighbor 172.16.6.254 family inet unicast

root@R6> show bgp summary group eBGP-AS64502 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 3 Peers: 3 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.rtarget.0        
                       1          1          0          0          0          0
inet.0               
                     147        144          0          0          0          0
inet6.0              
                      24          6          0          0          0          0
bgp.l3vpn.0          
                      14         14          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.6.254          64502      27603      30453       0       0 1w2d 14:00:39 Establ
  inet.0: 7/8/7/0
```

After establishing the session, we conducted a stress test. While running a continuous ping to the customer's network (201.2.1.1), we manually disabled one of the physical interfaces (ge-0/0/9).
```
root@R6> ping source 172.17.0.6 201.2.1.1      
PING 201.2.1.1 (201.2.1.1): 56 data bytes
64 bytes from 201.2.1.1: icmp_seq=0 ttl=64 time=45.351 ms
64 bytes from 201.2.1.1: icmp_seq=1 ttl=64 time=2.738 ms
64 bytes from 201.2.1.1: icmp_seq=2 ttl=64 time=350.220 ms
64 bytes from 201.2.1.1: icmp_seq=3 ttl=64 time=170.305 ms
64 bytes from 201.2.1.1: icmp_seq=4 ttl=64 time=7.789 ms
64 bytes from 201.2.1.1: icmp_seq=5 ttl=64 time=6.748 ms
64 bytes from 201.2.1.1: icmp_seq=6 ttl=64 time=8.145 ms
64 bytes from 201.2.1.1: icmp_seq=7 ttl=64 time=23.884 ms
64 bytes from 201.2.1.1: icmp_seq=8 ttl=64 time=244.605 ms
64 bytes from 201.2.1.1: icmp_seq=9 ttl=64 time=50.803 ms
64 bytes from 201.2.1.1: icmp_seq=10 ttl=64 time=8.790 ms
....
[edit interfaces ge-0/0/9]
root@R6# set disable 

[edit interfaces ge-0/0/9]
root@R6# top 

[edit]
root@R6# commit and-quit
....
64 bytes from 201.2.1.1: icmp_seq=11 ttl=64 time=5.862 ms
64 bytes from 201.2.1.1: icmp_seq=12 ttl=64 time=7.650 ms
64 bytes from 201.2.1.1: icmp_seq=13 ttl=64 time=46.283 ms
64 bytes from 201.2.1.1: icmp_seq=14 ttl=64 time=290.740 ms
64 bytes from 201.2.1.1: icmp_seq=15 ttl=64 time=5.517 ms
64 bytes from 201.2.1.1: icmp_seq=16 ttl=64 time=2.613 ms
64 bytes from 201.2.1.1: icmp_seq=17 ttl=64 time=4.472 ms
64 bytes from 201.2.1.1: icmp_seq=18 ttl=64 time=85.001 ms
64 bytes from 201.2.1.1: icmp_seq=19 ttl=64 time=16.531 ms
64 bytes from 201.2.1.1: icmp_seq=20 ttl=64 time=185.696 ms
64 bytes from 201.2.1.1: icmp_seq=21 ttl=64 time=11.961 ms
64 bytes from 201.2.1.1: icmp_seq=22 ttl=64 time=10.123 ms
64 bytes from 201.2.1.1: icmp_seq=23 ttl=64 time=170.741 ms
64 bytes from 201.2.1.1: icmp_seq=24 ttl=64 time=20.606 ms
64 bytes from 201.2.1.1: icmp_seq=25 ttl=64 time=44.506 ms
64 bytes from 201.2.1.1: icmp_seq=26 ttl=64 time=25.093 ms
64 bytes from 201.2.1.1: icmp_seq=27 ttl=64 time=9.144 ms
64 bytes from 201.2.1.1: icmp_seq=28 ttl=64 time=7.967 ms
64 bytes from 201.2.1.1: icmp_seq=29 ttl=64 time=4.193 ms
^C
--- 201.2.1.1 ping statistics ---
30 packets transmitted, 30 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.613/62.469/350.220/93.325 ms
```
The ping output shows that even during the interface shutdown, zero packets were lost. The static route immediately shifted all traffic to the remaining interface (ge-0/0/5), and the BGP session—anchored to the loopback—never flapped.

This is the power of a well-architected Multihop BGP session!

To wrap up our external peering configurations, we have Customer 1 (C1). This customer is dual-homed to our network via R6 and R7. Like C2, they are still IPv4-only.

For this customer, we have implemented several advanced stability and traffic engineering constraints:

Prefix Security: A strict limit of 20 prefixes. If the customer exceeds this, the session will automatically shut down (teardown).

Aggressive Dampening: To prevent unstable routes from affecting our core, we've defined a custom "aggressive" dampening profile.

Traffic Steering: We prefer R6 for both sending and receiving traffic. We'll use Local Preference for inbound control and MED for outbound influence.

With the constraints defined, let's go. 

First, we define our custom dampening parameters and the steering logic. On R7, we set a lower Local Preference with the value of 590 and a higher MED with the value of 10 to make it the secondary path.
```
set policy-options damping aggressive half-life 20
set policy-options damping aggressive reuse 500
set policy-options damping aggressive suppress 2500
set policy-options policy-statement aggressive-damp then damping aggressive
```
R6:
```
set policy-options as-path AS64501 .*64501
set policy-options policy-statement Entrada-C1 term deny-gw from as-path AS64501
set policy-options policy-statement Entrada-C1 term deny-gw from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Entrada-C1 term deny-gw then reject
set policy-options policy-statement Entrada-C1 term BH from as-path AS64501
set policy-options policy-statement Entrada-C1 term BH from community Blackhole
set policy-options policy-statement Entrada-C1 term BH from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-C1 term BH then next-hop discard
set policy-options policy-statement Entrada-C1 term BH then accept
set policy-options policy-statement Entrada-C1 term Baixa-LP from as-path AS64501
set policy-options policy-statement Entrada-C1 term Baixa-LP from community Cust-LP-90
set policy-options policy-statement Entrada-C1 term Baixa-LP from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-C1 term Baixa-LP then local-preference 90
set policy-options policy-statement Entrada-C1 term Baixa-LP then accept
set policy-options policy-statement Entrada-C1 term 1 from as-path AS64501
set policy-options policy-statement Entrada-C1 term 1 from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-C1 term 1 then local-preference 600
set policy-options policy-statement Entrada-C1 term 1 then community add C1
set policy-options policy-statement Entrada-C1 term 1 then community add Customer
set policy-options policy-statement Entrada-C1 term 1 then accept
set policy-options policy-statement Entrada-C1 then reject

set policy-options policy-statement Saida-C1 term Default from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Saida-C1 term Default then accept
set policy-options policy-statement Saida-C1 term General from community Provider
set policy-options policy-statement Saida-C1 term General from community Customer
set policy-options policy-statement Saida-C1 term General from community IX
set policy-options policy-statement Saida-C1 term General then accept
set policy-options policy-statement Saida-C1 term DCs from route-filter 172.17.0.0/22 exact
set policy-options policy-statement Saida-C1 term DCs then accept
set policy-options policy-statement Saida-C1 then reject
```
R7:
```
set policy-options as-path AS64501 .*64501
set policy-options policy-statement Entrada-C1 term deny-gw from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Entrada-C1 term deny-gw then reject
set policy-options policy-statement Entrada-C1 term BH from as-path AS64501
set policy-options policy-statement Entrada-C1 term BH from community Blackhole
set policy-options policy-statement Entrada-C1 term BH from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-C1 term BH then next-hop discard
set policy-options policy-statement Entrada-C1 term BH then accept
set policy-options policy-statement Entrada-C1 term Baixa-LP from as-path AS64501
set policy-options policy-statement Entrada-C1 term Baixa-LP from community Cust-LP-90
set policy-options policy-statement Entrada-C1 term Baixa-LP from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-C1 term Baixa-LP then local-preference 90
set policy-options policy-statement Entrada-C1 term Baixa-LP then accept
set policy-options policy-statement Entrada-C1 term 1 from as-path AS64501
set policy-options policy-statement Entrada-C1 term 1 from route-filter 0.0.0.0/0 prefix-length-range /8-/24
set policy-options policy-statement Entrada-C1 term 1 then local-preference 590
set policy-options policy-statement Entrada-C1 term 1 then community add C1
set policy-options policy-statement Entrada-C1 term 1 then community add Customer
set policy-options policy-statement Entrada-C1 term 1 then accept
set policy-options policy-statement Entrada-C1 then reject

set policy-options policy-statement Saida-C1 term Default from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Saida-C1 term Default then metric 10
set policy-options policy-statement Saida-C1 term Default then accept
set policy-options policy-statement Saida-C1 term General from community Provider
set policy-options policy-statement Saida-C1 term General from community Customer
set policy-options policy-statement Saida-C1 term General from community IX
set policy-options policy-statement Saida-C1 term General then metric 10
set policy-options policy-statement Saida-C1 term General then accept
set policy-options policy-statement Saida-C1 term DCs from route-filter 172.17.0.0/22 exact
set policy-options policy-statement Saida-C1 term DCs then metric 10
set policy-options policy-statement Saida-C1 term DCs then accept
set policy-options policy-statement Saida-C1 then reject
```

In the BGP group, we apply the damping via policy and set the prefix-limit. The teardown option is the "nuclear" choice—it drops the session immediately upon violation.
```
set protocols bgp group eBGP-AS64501 type external
set protocols bgp group eBGP-AS64501 description eBGP-AS64501
set protocols bgp group eBGP-AS64501 log-updown
set protocols bgp group eBGP-AS64501 damping
set protocols bgp group eBGP-AS64501 import aggressive-damp
set protocols bgp group eBGP-AS64501 import Entrada-C1
set protocols bgp group eBGP-AS64501 export Saida-C1
set protocols bgp group eBGP-AS64501 peer-as 64501
set protocols bgp group eBGP-AS64501 neighbor 172.16.6.2 family inet unicast

set protocols bgp group eBGP-AS64501 type external
set protocols bgp group eBGP-AS64501 description eBGP-AS64501
set protocols bgp group eBGP-AS64501 log-updown
set protocols bgp group eBGP-AS64501 damping
set protocols bgp group eBGP-AS64501 import aggressive-damp
set protocols bgp group eBGP-AS64501 import Entrada-C1
set protocols bgp group eBGP-AS64501 export Saida-C1
set protocols bgp group eBGP-AS64501 peer-as 64501
set protocols bgp group eBGP-AS64501 neighbor 172.16.7.2 family inet unicast prefix-limit maximum 20
set protocols bgp group eBGP-AS64501 neighbor 172.16.7.2 family inet unicast prefix-limit teardown
```
Let's check the results now:
```
root@R6> show bgp summary group eBGP-AS64501 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 3 Peers: 3 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.rtarget.0        
                       1          1          0          0          0          0
inet.0               
                     147        144          0          0          0          0
inet6.0              
                      24          6          0          0          0          0
bgp.l3vpn.0          
                      14         14          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.6.2            64501      27715      30593       0       0 1w2d 14:54:13 Establ
  inet.0: 8/8/8/0

root@R7> show bgp summary group eBGP-AS64501 
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
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.16.7.2            64501         16          4       0  346371           1 Active
```
During testing, the customer advertised 21 prefixes. As expected, the session on R7 moved to the "Active" (Idle) state and refused to establish. The logs confirm the violation:
```
Feb 25 11:29:35  R7 rpd[24610]: BGP_CEASE_PREFIX_LIMIT_EXCEEDED: 172.16.7.2 (External AS 64501): Shutting down peer due to exceeding configured maximum prefix-limit(20) for inet-unicast nlri: 21 (instance master)
```
Once the customer corrected their advertisement, we verified the path selection. A traceroute from the CE shows traffic correctly flowing towards R6 and R7:
```
[admin@C1-1] > tool traceroute 172.17.0.7 src-address=201.1.0.1
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS     LOSS  SENT  LAST   AVG  BEST  WORST  STD-DEV
1  172.16.6.1  0%       4  4.4ms  8.8  4.4   12.8   3.7    
2  172.17.0.7  0%       4  3.6ms  4.5  2.7   8.2    2.2

[admin@C1-1] > tool traceroute 172.17.0.6 src-address=201.1.0.1
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS     LOSS  SENT  LAST   AVG  BEST  WORST  STD-DEV
1  172.17.0.6  0%       2  0.8ms  0.8  0.8   0.8          0 
```
And, let's check if we are changing the local-pref value: We can't test the download teste because we don't have the iBGP defined yet. 
```
root@R6> show route 201.1.0.1 

inet.0: 225 destinations, 228 routes (220 active, 0 holddown, 5 hidden)
+ = Active Route, - = Last Active, * = Both

201.1.0.0/24       *[BGP/170] 1w2d 15:04:22, localpref 600
                      AS path: 64501 I, validation-state: unverified
                    >  to 172.16.6.2 via ge-0/0/8.601

root@R7> show route 201.1.0.1 

inet.0: 118 destinations, 119 routes (113 active, 0 holddown, 5 hidden)
+ = Active Route, - = Last Active, * = Both

201.1.0.0/24       *[BGP/170] 00:03:29, localpref 590
                      AS path: 64501 I, validation-state: unverified
                    >  to 172.16.7.2 via ge-0/0/5.701
```
We also checked the Aggressive Dampening status. Even though the route is currently active, Junos tracks the "Merit" and "Flaps" history, ready to suppress the prefix if it starts flapping again:
```
root@R7> show route damping decayed detail 

inet.0: 118 destinations, 119 routes (113 active, 0 holddown, 5 hidden)
201.1.0.0/24 (1 entry, 1 announced)
        *BGP    Preference: 170/-591
                Next hop type: Router, Next hop index: 672
                Address: 0x80ac594
                Next-hop reference count: 14, Next-hop session id: 348
                Kernel Table Id: 0
                Source: 172.16.7.2
                Next hop: 172.16.7.2 via ge-0/0/5.701, selected
                Session Id: 348
                State: <Active Ext>
                Local AS: 65020 Peer AS: 64501
                Age: 29 
                Validation State: unverified 
                Task: BGP_64501.172.16.7.2
                Announcement bits (2): 0-KRT 7-BGP_RT_Background 
                AS path: 64501 I 
                Communities: 65020:51 65020:64501
                Accepted
                Localpref: 590
                Router ID: 201.1.7.1
                Merit (last update/now): 1988/1965
                damping-parameters: aggressive
                Last update: 00000000:00:29 First update: 00000000:00:42
                Flaps: 2
                Thread: junos-main 
```
And... it's working, everything is ok!!!

We have now completed our eBGP peering, achieving all our objectives. I hope you will follow the entire chapter to prepare with me for JNCIE-SP.

In the next chapter, we will create our iBGP network and finally be able to test connectivity between the ASs! See you soon!
