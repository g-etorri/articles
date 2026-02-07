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
| R3 | ge-0/0/5.302 | 172.16.3.5/30 | N/A | 302 | Provider 3 | 65503 |
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
The IX-1 have the IP 192.168.12.253 and the IX-2 192.168.12.254. Here we have a tip to give to you, always mark a community in the import policy, this way, we can handle the routes imported by this peering easily. 
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
In this step, we can make the blackhole community with a regex, this way we can inform the customer to export the routes with his AS:666, and our routers will discard these routes.
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
And... This is OK!!! We have much more peerings to configure, let's go. 


