# L3VPN Configuration

What's up fellas. Today we'll start to delivery some services to our customers!!! We'll start with L3VPNs. 

The topology you already... No!!! This time we have a new topology image! 
<img width="1146" height="822" alt="image" src="https://github.com/user-attachments/assets/d7e82fbe-fd3a-4abb-872e-a6ff15d2a601" />

It's time to enable the family inet-vpn in our backbone! Let's add the family in both cluster of RR:
```
set protocols bgp group iBGP-AS65020-West family inet-vpn unicast nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-West family inet-vpn unicast no-install
set protocols bgp group iBGP-AS65020-East family inet-vpn unicast nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-East family inet-vpn unicast no-install
```
And enable the family in the clients:
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
  inet.0: 31/91/91/0
  inet6.0: 4/18/18/0
  bgp.l3vpn.0: 0/0/0/0
10.0.0.2              65020         46         73       0       0        1:31 Establ
  inet.0: 1/91/91/0
  inet6.0: 14/18/18/0
  bgp.l3vpn.0: 0/0/0/0
10.0.0.3              65020         20         73       0       0        1:31 Establ
  inet.0: 63/65/65/0
  inet6.0: 3/3/3/0
  bgp.l3vpn.0: 0/0/0/0
10.0.0.4              65020         19         74       0       0        1:31 Establ
  inet.0: 1/13/13/0
  inet6.0: 0/0/0/0
  bgp.l3vpn.0: 0/0/0/0
10.0.0.5              65020         15         77       0       0        1:31 Establ
  inet.0: 8/20/20/0
  inet6.0: 1/1/1/0
  bgp.l3vpn.0: 0/0/0/0
10.0.0.6              65020         14         73       0       0        1:31 Establ
  inet.0: 15/15/15/0
  inet6.0: 0/0/0/0
  bgp.l3vpn.0: 0/0/0/0
10.0.0.7              65020         17         79       0       0        1:31 Establ
  inet.0: 0/1/1/0
  inet6.0: 0/0/0/0                      
  bgp.l3vpn.0: 0/0/0/0
10.0.0.8              65020         13         78       0       0        1:31 Establ
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

