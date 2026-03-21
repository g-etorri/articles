# L3VPN Configuration

What's up fellas. Today we'll start to delivery some services to our customers!!! We'll start with L3VPNs. 

The topology you already... No!!! This time we have a new topology image! 
<img width="1146" height="822" alt="image" src="https://github.com/user-attachments/assets/d7e82fbe-fd3a-4abb-872e-a6ff15d2a601" />

It's time to enable the family inet-vpn in our backbone! Let's add the family in both cluster of RR. 
```
set protocols bgp group iBGP-AS65020-West family inet-vpn unicast nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-West family inet-vpn unicast no-install
set protocols bgp group iBGP-AS65020-East family inet-vpn unicast nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-East family inet-vpn unicast no-install
```
And, to optimize the advertisements, we can include the family route-target also! We can include the advertise-default knob in the RR to receive all RTs, but the clients will receive only the desired RTs. 
```
set protocols bgp group iBGP-AS65020-West family route-target advertise-default
set protocols bgp group iBGP-AS65020-East family route-target advertise-default
```
And enable the families in the clients:
```

root@RR> show bgp summary     
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 8 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0               
                     327        150          0          0          0          0
inet6.0              
                      41         23          0          0          0          0
bgp.l3vpn.0          
                      52         52          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.1              65020         46         58       0       0        1:31 Establ
  bgp.rtarget.0: 0/0/0/0
  inet.0: 31/91/91/0
  inet6.0: 4/18/18/0
  bgp.l3vpn.0: 0/0/0/0
10.0.0.2              65020         46         73       0       0        1:31 Establ
  bgp.rtarget.0: 0/0/0/0
  inet.0: 1/91/91/0
  inet6.0: 14/18/18/0
  bgp.l3vpn.0: 0/0/0/0
10.0.0.3              65020         20         73       0       0        1:31 Establ
  bgp.rtarget.0: 0/0/0/0
  inet.0: 63/65/65/0
  inet6.0: 3/3/3/0
  bgp.l3vpn.0: 0/0/0/0
10.0.0.4              65020         19         74       0       0        1:31 Establ
  bgp.rtarget.0: 0/0/0/0
  inet.0: 1/13/13/0
  inet6.0: 0/0/0/0
  bgp.l3vpn.0: 0/0/0/0
10.0.0.5              65020         15         77       0       0        1:31 Establ
  bgp.rtarget.0: 0/0/0/0
  inet.0: 8/20/20/0
  inet6.0: 1/1/1/0
  bgp.l3vpn.0: 0/0/0/0
10.0.0.6              65020         14         73       0       0        1:31 Establ
  bgp.rtarget.0: 0/0/0/0
  inet.0: 15/15/15/0
  inet6.0: 0/0/0/0
  bgp.l3vpn.0: 0/0/0/0
10.0.0.7              65020         17         79       0       0        1:31 Establ
  bgp.rtarget.0: 0/0/0/0
  inet.0: 0/1/1/0
  inet6.0: 0/0/0/0                      
  bgp.l3vpn.0: 0/0/0/0
10.0.0.8              65020         13         78       0       0        1:31 Establ
  bgp.rtarget.0: 0/0/0/0
  inet.0: 31/31/31/0
  inet6.0: 1/1/1/0
  bgp.l3vpn.0: 0/0/0/0
```
Ok, our BGP mesh is prepared to delivery the L3VPN services!!!

First, let's made the boriest step of the topology. Configure all interfaces accordingly this table:
| Router | Interface    | IP Address    |  IPv6 Address             | Customer    |
| ------ | ------------ | ------------- | ------------------------- | ----------- |
| R1     | ge-0/0/8.200 | 10.2.1.1/30   |                           | C2-Hub      |
| R1     | ge-0/0/8.201 | 10.2.1.5/30   |                           | C2-Spoke    |
| R1     | ge-0/0/8.202 | 10.2.1.9/30   |                           | C2-Internet |
| R1     | lo0.1        | 10.2.1.254/32 |                           | C2-Hub      |
| R1     | lo0.2        | 10.2.1.253/32 |                           | C2-Spoke    |
| R2     | ge-0/0/7.200 | 10.2.2.1/30   |                           | C2-Hub      |
| R2     | ge-0/0/7.201 | 10.2.2.5/30   |                           | C2-Spoke    |
| R2     | ge-0/0/7.202 | 10.2.2.9/30   |                           | C2-Internet |
| R2     | lo0.1        | 10.2.2.254/32 |                           | C2-Hub      |
| R2     | lo0.2        | 10.2.2.253/32 |                           | C2-Spoke    |
| R3     | ge-0/0/1.300 |               | fc09:c0:ffee:3:3::1/126   | C3          |
| R3     | ge-0/0/8.100 | 10.1.3.1/30   |                           | C1          |
| R3     | lo0.1        | 10.1.3.254/32 |                           | C1          |
| R3     | lo0.2        | 10.3.3.254/32 | fc09:c0:ffee:3:3::254/128 | C3          |
| R4     | ge-0/0/8.100 | 10.1.4.1/30   |                           | C1          |
| R4     | ge-0/0/9.201 | 10.2.4.1/30   |                           | C2          |
| R4     | lo0.1        | 10.1.4.254/32 |                           | C1          |
| R4     | lo0.2        | 10.2.4.254/32 |                           | C2          |
| R5     | ge-0/0/7.201 | 10.2.5.1/30   |                           | C2          |
| R5     | lo0.1        | 10.2.5.254/32 |                           | C2          |
| R6     | ge-0/0/4.100 | 10.1.6.1/30   |                           | C1          |
| R6     | lo0.1        | 10.1.6.254/32 |                           | C1          |
| R7     | ge-0/0/6.201 | 10.2.7.1/30   |                           | C2          |
| R7     | lo0.1        | 10.2.7.254/32 |                           | C2          |
| R8     | ge-0/0/6.100 | 10.1.8.1/30   |                           | C1          |
| R8     | ge-0/0/8.300 |               | fc09:c0:ffee:3:8::1/126   | C3          |
| R8     | lo0.1        | 10.1.8.254/32 |                           | C1          |
| R8     | lo0.2        | 10.3.8.254/32 | fc09:c0:ffee:3:8::254/128 | C3          |

Here we're following a standard for the point-to-point networks. For the IPv4 network, the model is 10.X.Y.0/24, where X is the customer number and Y the router number, and the last octet is ordenated according to use. 
For the IPv6 networks, we'll use a similar model, fc09:c0:ffee:X:Y::/80, where X is the customer number and Y the router number, and the last octet is ordenated according to use. 

Ok, with this defined, let's make the configuration. I will use the R1 as example again, and we can apply similarly on the other routers. 
```
set interfaces ge-0/0/8 description to-C2-1
set interfaces ge-0/0/8 flexible-vlan-tagging
set interfaces ge-0/0/8 encapsulation flexible-ethernet-services
set interfaces ge-0/0/8 unit 200 description to-C2-HUB
set interfaces ge-0/0/8 unit 200 vlan-id 200
set interfaces ge-0/0/8 unit 200 family inet address 10.2.1.1/30
set interfaces ge-0/0/8 unit 201 description to-C2-SPOKE
set interfaces ge-0/0/8 unit 201 vlan-id 201
set interfaces ge-0/0/8 unit 201 family inet address 10.2.1.5/30
set interfaces ge-0/0/8 unit 202 description to-C2-INTERNET
set interfaces ge-0/0/8 unit 202 vlan-id 202
set interfaces ge-0/0/8 unit 202 family inet address 10.2.1.9/30
set interfaces lo0 unit 1 family inet address 10.2.1.254/32
set interfaces lo0 unit 2 family inet address 10.2.1.253/32
```

Ok, now I'll explore each customer individually. We have 3 L3VPN customers today

To keep the organization, before all the configuration let's define the RTs and RDs models. I'll use the identificator 100 for Customer 1, 200 for Customer 2 and 300 for Customer 3. For RDs we'll adopt the RD type 2, this is the best RD! And for RTs, we'll use our ASN. 

For example, Customer 1 at R8. The RD will be 10.0.0.8:100 and the RT will be target:65020:100, got it? I'm sure yes. 

The Customer 1, this customer wants a full mesh communication and all the routers inside the same OSPF area. 
| Customer| Site | Router   | PE-CE Protocol  | Protocol Details      |
| ------- | ---- | -------- | --------------- | --------------------- |
| C1      | S1   | CE1-1    | OSPF            | Area 0.0.0.0          |
| C1      | S2   | CE1-2    | OSPF            | Area 0.0.0.0          |
| C1      | S2   | CE1-3    | OSPF            | Area 0.0.0.0          |
| C1      | S3   | CE1-4    | OSPF            | Area 0.0.0.0          |

This customer have the Site 1 connected in R8, Site 3 connected in R6 and in his Site 2, customer haves two routers CE1-2 connected in R3 and the CE1-3 connected in R4, between these two routers there is a OSPF link, and we need to ensure if this link fails, the customer can use our backbone to the communicate these sites. 

So, the lore is explained. Let's go to the configuration. 

First, let's start at R8. We need to create the VRF, bind the interface at it and establish the OSPF with de CE1-1. 
```
set routing-instances VRF-C1 instance-type vrf
set routing-instances VRF-C1 protocols ospf area 0.0.0.0 interface ge-0/0/6.100 interface-type p2p
set routing-instances VRF-C1 protocols ospf area 0.0.0.0 interface lo0.1 passive
set routing-instances VRF-C1 description VRF-C1
set routing-instances VRF-C1 interface ge-0/0/6.100
set routing-instances VRF-C1 interface lo0.1
set routing-instances VRF-C1 route-distinguisher 10.0.0.8:100
set routing-instances VRF-C1 vrf-target target:65020:100
set routing-instances VRF-C1 vrf-table-label
```
The vrf-table-label knob is used to define a label without have a next-hop in the VRF. This is perfect to efford resources, and to troubleshooting, considering that we have a loopback defined into VRF.

Now, let's repeat this configuration similarly in the other routers. I'll omit here to not extend this doc. 

Let's check the adjacency: 
```
root@R8> show ospf neighbor instance VRF-C1 detail 
Address          Interface              State           ID               Pri  Dead
10.1.8.2         ge-0/0/6.100           Full            10.1.0.1         128    38
  Area 0.0.0.0, opt 0x2, DR 0.0.0.0, BDR 0.0.0.0
  Up 1w0d 02:10:10, adjacent 1w0d 02:10:10
```
Everything is good so far. Now, to permit the communication trough the L3VPN, we need to export the BGP routes from the other sites. But, if we do this we'll have a little problem that the customer have pointed, our router will flood these routes as LSAs type 3. 
To avoid this, we can use an OSPF feature, the sham-links!!!

With sham-links, we can establish a virtual-adjacency between our backbone routers inside the VRF, and for the customer the network will be a simple OSPF domain. 

To do this, we only need to define the local address and the remote-address of the sham-link. Let's configure the R8 and similarly the another ones. 
```
set routing-instances VRF-C1 protocols ospf sham-link local 10.1.8.254
set routing-instances VRF-C1 protocols ospf area 0.0.0.0 sham-link-remote 10.1.3.254
set routing-instances VRF-C1 protocols ospf area 0.0.0.0 sham-link-remote 10.1.4.254
set routing-instances VRF-C1 protocols ospf area 0.0.0.0 sham-link-remote 10.1.6.254
```
Let's verify if we have established the adjacencies:
```
root@R8> show ospf neighbor instance VRF-C1           
Address          Interface              State           ID               Pri  Dead
10.1.8.2         ge-0/0/6.100           Full            10.1.0.1         128    39
10.1.3.254       shamlink.0             Full            10.1.3.254         0    34
10.1.4.254       shamlink.1             Full            10.1.4.254         0    38
10.1.6.254       shamlink.2             Full            10.1.6.254         0    37
```
Ok, everything looks good, now let's verify the OSPF database:
```
root@R8> show ospf database instance VRF-C1 detail 

    OSPF database, Area 0.0.0.0
 Type       ID               Adv Rtr           Seq      Age  Opt  Cksum  Len 
Router   10.1.0.1         10.1.0.1         0x80000158   743  0x2  0xb9c4  60
  bits 0x0, link count 3
  id 10.1.8.254, data 10.1.8.2, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.0.1, data 255.255.255.255, Type Stub (3)
    Topology count: 0, Default metric: 1
  id 10.1.8.0, data 255.255.255.252, Type Stub (3)
    Topology count: 0, Default metric: 1
  Topology default (ID 0)
    Type: PointToPoint, Node ID: 10.1.8.254
      Metric: 1, Bidirectional

Router   10.1.0.2         10.1.0.2         0x80000158   256  0x2  0xf154  72
  bits 0x0, link count 4
  id 10.1.0.3, data 10.1.23.1, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.0.2, data 255.255.255.255, Type Stub (3)
    Topology count: 0, Default metric: 1
   id 10.1.3.0, data 255.255.255.252, Type Stub (3)
    Topology count: 0, Default metric: 1
  id 10.1.23.0, data 255.255.255.252, Type Stub (3)
    Topology count: 0, Default metric: 1
  Topology default (ID 0)
    Type: PointToPoint, Node ID: 10.1.0.3
      Metric: 1, Bidirectional

Router   10.1.0.3         10.1.0.3         0x80000158  1632  0x2  0x5cb7  84
  bits 0x0, link count 5
  id 10.1.0.2, data 10.1.23.2, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.4.254, data 10.1.4.2, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.0.3, data 255.255.255.255, Type Stub (3)
    Topology count: 0, Default metric: 1
  id 10.1.4.0, data 255.255.255.252, Type Stub (3)
    Topology count: 0, Default metric: 1
  id 10.1.23.0, data 255.255.255.252, Type Stub (3)
    Topology count: 0, Default metric: 1
  Topology default (ID 0)
    Type: PointToPoint, Node ID: 10.1.4.254
      Metric: 1, Bidirectional
    Type: PointToPoint, Node ID: 10.1.0.2
      Metric: 1, Bidirectional

Router   10.1.0.4         10.1.0.4         0x80000158   910  0x2  0x5d1e  60
  bits 0x0, link count 3
  id 10.1.6.254, data 10.1.6.2, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.0.4, data 255.255.255.255, Type Stub (3)
    Topology count: 0, Default metric: 1
  id 10.1.6.0, data 255.255.255.252, Type Stub (3)
    Topology count: 0, Default metric: 1
  Topology default (ID 0)
    Type: PointToPoint, Node ID: 10.1.6.254
      Metric: 1, Bidirectional

Router   10.1.3.254       10.1.3.254       0x800000d2   751  0x22 0xa0b4  72
  bits 0x1, link count 4
  id 10.1.3.0, data 255.255.255.252, Type Stub (3)
    Topology count: 0, Default metric: 1
  id 10.1.4.254, data 128.1.0.0, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.6.254, data 128.1.0.1, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.8.254, data 128.1.0.2, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  Topology default (ID 0)
    Type: PointToPoint, Node ID: 10.1.8.254
      Metric: 1, Bidirectional
    Type: PointToPoint, Node ID: 10.1.6.254
      Metric: 1, Bidirectional          
    Type: PointToPoint, Node ID: 10.1.4.254
      Metric: 1, Bidirectional

Router   10.1.4.254       10.1.4.254       0x800000d4   189  0x22 0x80a3  84
  bits 0x1, link count 5
  id 10.1.0.3, data 10.1.4.1, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.4.0, data 255.255.255.252, Type Stub (3)
    Topology count: 0, Default metric: 1
  id 10.1.3.254, data 128.1.0.0, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.6.254, data 128.1.0.1, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.8.254, data 128.1.0.2, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  Topology default (ID 0)
    Type: PointToPoint, Node ID: 10.1.8.254
      Metric: 1, Bidirectional
    Type: PointToPoint, Node ID: 10.1.6.254
      Metric: 1, Bidirectional
    Type: PointToPoint, Node ID: 10.1.3.254
      Metric: 1, Bidirectional
    Type: PointToPoint, Node ID: 10.1.0.3
      Metric: 1, Bidirectional
 
Router   10.1.6.254       10.1.6.254       0x800000d3   747  0x22 0x52cb  84
  bits 0x1, link count 5
  id 10.1.0.4, data 10.1.6.1, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.6.0, data 255.255.255.252, Type Stub (3)
    Topology count: 0, Default metric: 1
  id 10.1.3.254, data 128.1.0.0, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.4.254, data 128.1.0.1, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.8.254, data 128.1.0.2, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  Topology default (ID 0)
    Type: PointToPoint, Node ID: 10.1.8.254
      Metric: 1, Bidirectional
    Type: PointToPoint, Node ID: 10.1.4.254
      Metric: 1, Bidirectional
    Type: PointToPoint, Node ID: 10.1.3.254
      Metric: 1, Bidirectional
    Type: PointToPoint, Node ID: 10.1.0.4
      Metric: 1, Bidirectional

Router  *10.1.8.254       10.1.8.254       0x800000d3   742  0x22 0xe139  84
  bits 0x1, link count 5                
  id 10.1.0.1, data 10.1.8.1, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.8.0, data 255.255.255.252, Type Stub (3)
    Topology count: 0, Default metric: 1
  id 10.1.3.254, data 128.1.0.0, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.4.254, data 128.1.0.1, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  id 10.1.6.254, data 128.1.0.2, Type PointToPoint (1)
    Topology count: 0, Default metric: 1
  Topology default (ID 0)
    Type: PointToPoint, Node ID: 10.1.6.254
      Metric: 1, Bidirectional
    Type: PointToPoint, Node ID: 10.1.4.254
      Metric: 1, Bidirectional
    Type: PointToPoint, Node ID: 10.1.3.254
      Metric: 1, Bidirectional
    Type: PointToPoint, Node ID: 10.1.0.1
      Metric: 1, Bidirectional
```
Mission accomplished!!! 

Oh... wait. Customer is calling me, now he asked to have internet trough site 2 without changes in his network. 

No... now we need to make a ugly change. We need to export a default route into OSPF, and leak the loopback addresses of site 2 to our inet.0. Let's do it. 

To have a default route in our VRF, let's create this setting to inet.0 table. 
```
set routing-instances VRF-C1 routing-options static route 0.0.0.0/0 next-table inet.0
```
And, to inject this route into OSPF, we need to do it with a route-policy. 
```
set policy-options policy-statement redistribute-ospf-vrf-c1 term default from route-filter 0.0.0.0/0 exact
set policy-options policy-statement redistribute-ospf-vrf-c1 term default then accept

set routing-instances VRF-C1 protocols ospf export redistribute-ospf-vrf-c1
```
Ok, now we are redistritibuting this route into OSPF correctly. But, we don't have internet access yet. 

We need to leak these routes into our inet.0, and export this prefix to our peerings. 

First, let's leak this routes into inet.0 trough a rib-group. To create the rib-group, first we set the source rib, then the destination rib, and to leak the OSPF routes we need to apply this rib-group into OSPF configuration. 
```
set routing-options rib-groups leak-c1-routes import-rib [ VRF-C1.inet.0 inet.0 ]

set routing-instances VRF-C1 protocols ospf rib-groups inet leak-c1-routes
```
Now, if we check the inet.0, we already have the routes of site 2. 
```
root@R4> show route table inet.0 10.1.0.0/24    

inet.0: 231 destinations, 237 routes (224 active, 0 holddown, 7 hidden)
+ = Active Route, - = Last Active, * = Both

10.1.0.2/32        *[OSPF/10] 00:01:19, metric 3
                    >  to 10.1.4.2 via ge-0/0/8.100
10.1.0.3/32        *[OSPF/10] 00:01:19, metric 2
                    >  to 10.1.4.2 via ge-0/0/8.100
```
Ok, with this, we are ready to provide internet access to this customer. To make this things pretty, let's create an aggregate prefix to export for our peering correctly, and we'll mark this prefix as customer prefix, as the other peering customer prefixes. 
```
set routing-options aggregate route 10.1.0.0/24 as-path aggregator 65020 10.0.0.4
set routing-options aggregate route 10.1.0.0/24 discard

set policy-options policy-statement Saida-RR term 1 from protocol bgp
set policy-options policy-statement Saida-RR term 1 then accept
set policy-options policy-statement Saida-RR term 2 from protocol aggregate
set policy-options policy-statement Saida-RR term 2 from route-filter 10.1.0.0/24 exact
set policy-options policy-statement Saida-RR term 2 then community add Customer
set policy-options policy-statement Saida-RR term 2 then accept
```
This way, our customer will have internet access trough R4. Let's ask a test to the customer now:
```
[admin@CE1-2] > tool traceroute src-address=10.1.0.2 201.1.0.1
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS      LOSS  SENT  LAST    AVG   BEST  WORST  STD-DEV
1  10.1.23.2    0%       3  0.6ms   0.5   0.5   0.6    0      
2  10.1.4.1     0%       3  16.8ms  12.8  4.5   17     5.8    
3  10.200.0.17  0%       3  22ms    41    22    53     13.6   
4  201.1.0.1    0%       3  14.4ms  15.1  13    17.8   2    
```
And... everything looks ok! But, we need to ensure the internet access if the R4 fails, so, we need to replicate this configuration on R3 to ensure the redundancy. You can do it, right? I trust you. 

Now, let's go for the Customer 2! 

Customer 2 wants a hub and spoke topology. Where the Site 1 is a hub, composed of two routers and all the network will have internet access trough the Site 1. This is a classic L3VPN structure. 

First of all, let's understand the methodology. We'll have two VRFs configured (HUB and SPOKE) in the routers connected to the hub, and three links. 
* Internet link, this link will provide internet access for the site.
* VRF-HUB link, trough this link the PEs will import the routes from hub, without accept routes from SPOKE PEs. The SPOKE PEs will accept the routes with this RT. Customer ask us to not receive the routes from the other hub routers, because he haves a backdoor link. 
* VRF-SPOKE link, trough this link the PEs will export the routes to hub, without accept routes from the hub routers. The SPOKE PEs will export the routes with this RT.

This way, we can organize the spoke prefixes, and hub prefixes. 

First, let's establish the internet access to the customer, this will be a common BGP peering with him. We'll export only a default route and accept the 10.2.0.0/24 prefix from customer. 
R1:
```
set policy-options policy-statement Entrada-CE2 term 1 from route-filter 10.2.0.0/24 exact
set policy-options policy-statement Entrada-CE2 term 1 then accept
set policy-options policy-statement Entrada-CE2 term 1 then community add Customer
set policy-options policy-statement Entrada-CE2 then reject

set policy-options policy-statement Saida-CE2 term 1 from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Saida-CE2 term 1 then accept
set policy-options policy-statement Saida-CE2 then reject

set protocols bgp group eBGP-C2-AS64702 type external
set protocols bgp group eBGP-C2-AS64702 description eBGP-C2-AS64702
set protocols bgp group eBGP-C2-AS64702 import Entrada-CE2
set protocols bgp group eBGP-C2-AS64702 export Saida-CE2
set protocols bgp group eBGP-C2-AS64702 peer-as 64702
set protocols bgp group eBGP-C2-AS64702 neighbor 10.2.1.10 family inet unicast
```
R2:
```
set policy-options policy-statement Entrada-CE2 term 1 from route-filter 10.2.0.0/24 exact
set policy-options policy-statement Entrada-CE2 term 1 then accept
set policy-options policy-statement Entrada-CE2 term 1 then community add Customer
set policy-options policy-statement Entrada-CE2 then reject

set policy-options policy-statement Saida-CE2 term 1 from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Saida-CE2 term 1 then accept
set policy-options policy-statement Saida-CE2 then reject

set protocols bgp group eBGP-CE2-AS64702 type external
set protocols bgp group eBGP-CE2-AS64702 description eBGP-CE2-AS64702
set protocols bgp group eBGP-CE2-AS64702 import Entrada-CE2
set protocols bgp group eBGP-CE2-AS64702 export Saida-CE2
set protocols bgp group eBGP-CE2-AS64702 peer-as 64702
set protocols bgp group eBGP-CE2-AS64702 neighbor 10.2.2.10 family inet unicast
```
Let's check the stats of the BGP session!! 

```
root@R1> show bgp summary group eBGP-C2-AS64702       
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 5 Peers: 6 Down peers: 0
...
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.2.1.10             64702      43528      47600       0       0 2w1d 2:40:10 Establ
  inet.0: 1/1/1/0

root@R1> show route receive-protocol bgp 10.2.1.10 table inet.0 

inet.0: 224 destinations, 247 routes (220 active, 0 holddown, 5 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.2.0.0/24             10.2.1.10                               64702 I

root@R1> show route advertising-protocol bgp 10.2.1.10 table inet.0 

inet.0: 224 destinations, 247 routes (220 active, 0 holddown, 5 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 0.0.0.0/0               Self                                    I
..........
root@R2> show bgp summary group eBGP-CE2-AS64702                   
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 5 Peers: 6 Down peers: 0
...
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.2.2.10             64702      43529      47601       0       0 2w1d 2:40:48 Establ
  inet.0: 1/1/1/0

root@R2> show route receive-protocol bgp 10.2.2.10 table inet.0    

inet.0: 225 destinations, 272 routes (220 active, 0 holddown, 6 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.2.0.0/24             10.2.2.10                               64702 I

root@R2> show route advertising-protocol bgp 10.2.2.10 table inet.0 

inet.0: 225 destinations, 272 routes (220 active, 0 holddown, 6 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 0.0.0.0/0               Self                                    I

```
Everything looks ok to the customer access the internet!!! Let's verify this access later. 

Now, let's go to the L3VPN! 

First, let's pin the details of the BGP connections. 
| Customer | Site | Router | PE-CE Protocol | Protocol Details |
| ------- | ---- | -------- | --------------- | --------------------- |
| C2      | S1   | CE2-1    | BGP             | AS64702               |
| C2      | S1   | CE2-2    | BGP             | AS64702               |
| C2      | S2   | CE2-3    | BGP             | AS64702               |
| C2      | S2   | CE2-4    | BGP             | AS64702               |
| C2      | S3   | CE2-5    | BGP             | AS64702               |

Now, let's start configuring the HUB-VRF! This VRF will be configured only on the R1 and R2, and will only receive the hub routes. 
R1:
```
set routing-options autonomous-system loops 3

set routing-instances VRF-C2-HUB instance-type vrf
set routing-instances VRF-C2-HUB protocols bgp group eBGP-CE2-1-HUB type external
set routing-instances VRF-C2-HUB protocols bgp group eBGP-CE2-1-HUB description eBGP-CE2-1-HUB
set routing-instances VRF-C2-HUB protocols bgp group eBGP-CE2-1-HUB export deny-all
set routing-instances VRF-C2-HUB protocols bgp group eBGP-CE2-1-HUB peer-as 64702
set routing-instances VRF-C2-HUB protocols bgp group eBGP-CE2-1-HUB as-override
set routing-instances VRF-C2-HUB protocols bgp group eBGP-CE2-1-HUB neighbor 10.2.1.2 family inet unicast
set routing-instances VRF-C2-HUB description VRF-C2-HUB
set routing-instances VRF-C2-HUB interface ge-0/0/8.200
set routing-instances VRF-C2-HUB interface lo0.1
set routing-instances VRF-C2-HUB route-distinguisher 10.0.0.1:200
set routing-instances VRF-C2-HUB vrf-target target:65020:200
set routing-instances VRF-C2-HUB vrf-table-label

set policy-options policy-statement deny-all then reject
```
R2:
```
set routing-options autonomous-system loops 3

set routing-instances VRF-C2-HUB instance-type vrf
set routing-instances VRF-C2-HUB protocols bgp group eBGP-CE2-2-HUB type external
set routing-instances VRF-C2-HUB protocols bgp group eBGP-CE2-2-HUB description eBGP-CE2-2-HUB
set routing-instances VRF-C2-HUB protocols bgp group eBGP-CE2-2-HUB export deny-all
set routing-instances VRF-C2-HUB protocols bgp group eBGP-CE2-2-HUB peer-as 64702
set routing-instances VRF-C2-HUB protocols bgp group eBGP-CE2-2-HUB as-override
set routing-instances VRF-C2-HUB protocols bgp group eBGP-CE2-2-HUB neighbor 10.2.2.2 family inet unicast
set routing-instances VRF-C2-HUB description VRF-C2-HUB
set routing-instances VRF-C2-HUB interface ge-0/0/7.200
set routing-instances VRF-C2-HUB interface lo0.1
set routing-instances VRF-C2-HUB route-distinguisher 10.0.0.2:200
set routing-instances VRF-C2-HUB vrf-target target:65020:200
set routing-instances VRF-C2-HUB vrf-table-label

set policy-options policy-statement deny-all then reject
```
As I said, we won't export prefixes trough this session. Only receive the routes to export to the spoke sites, so I made an policy to reject all. 
Note: Here we can see two different knobs used: 
* AS Loops, we are using this to receive the default route in the VRF-HUB, this way the Customer can export the default route received from us to the spoke sites.
* as-override, this knob is used to avoid routes rejected by AS-LOOP both on the spoke sites and hub sites. Think with me, if we receive the loopback address from HUB, when we export this route to a spoke site, we'll export this route with the AS64702 on th AS-PATH, and the spoke have this AS also, so, to avoid loops this route could be rejected. With the as-override, we are removing the customer AS to handle this behavior, the AS-PATH will contain only our AS, the AS65020!

Now, let's check the sessions and what we have in the VRF-HUB! 
```
root@R1> show bgp summary group eBGP-CE2-1-HUB  
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 5 Peers: 6 Down peers: 0
...
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.2.1.2              64702      43546      47626       0       0 2w1d 2:51:00 Establ
  VRF-C2-HUB.inet.0: 3/3/3/0

root@R1> show route receive-protocol bgp 10.2.1.2 table VRF-C2-HUB.inet.0 

VRF-C2-HUB.inet.0: 8 destinations, 11 routes (8 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 0.0.0.0/0               10.2.1.2                                64702 65020 I
* 10.2.0.1/32             10.2.1.2                                64702 I
* 10.2.0.2/32             10.2.1.2                                64702 I

root@R2> show bgp summary group eBGP-CE2-2-HUB    
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 5 Peers: 6 Down peers: 0
...
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.2.2.2              64702      43543      47629       0       0 2w1d 2:51:44 Establ
  VRF-C2-HUB.inet.0: 3/3/3/0

root@R2> show route receive-protocol bgp 10.2.2.2 table VRF-C2-HUB.inet.0 

VRF-C2-HUB.inet.0: 8 destinations, 11 routes (8 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 0.0.0.0/0               10.2.2.2                                64702 65020 I
* 10.2.0.1/32             10.2.2.2                                64702 I
* 10.2.0.2/32             10.2.2.2                                64702 I
```
We are receiving a route with our own AS in the AS-PATH, so the loops are working!!! 

Now, let's make the VRF-SPOKE in the R1 and R2, this still be simple. In the SPOKE PEs will be different. 
```

```
