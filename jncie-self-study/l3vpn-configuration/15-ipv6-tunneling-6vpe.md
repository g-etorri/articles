# Tunneling IPv6 via 6VPE

Hello guys! Today, we'll continue to configure the L3VPN services of our network. 

You already know the topology. 
<img width="1146" height="822" alt="image" src="https://github.com/user-attachments/assets/d7e82fbe-fd3a-4abb-872e-a6ff15d2a601" />

In the last article, we didn't configure the Customer 3 VPN. Customer 3 has a different topology, the unique customer to have IPv6! So, this VPN is not inet-vpn, that's a inet6-vpn, or most knownly, 6VPE! 

To prepare our network to run 6VPE, we need to configure this MP-BGP family on the RR and on the PEs:
```
set protocols bgp group iBGP-AS65020-West family inet6-vpn unicast nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-West family inet6-vpn unicast no-install
set protocols bgp group iBGP-AS65020-East family inet6-vpn unicast nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-East family inet6-vpn unicast no-install
```
Note: If you remember the previous articles, we are using the no-install and no-resolution knobs to be easy as RR out-of-path, this is not a recomendation to use in real cases. 

Let's check if the sessions are established:
```
root@RR> show bgp summary 
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 8 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
bgp.rtarget.0        
                      32         15          0          0          0          0
inet.0               
                     389        153          0          0          0          0
inet6.0              
                      61         25          0          0          0          0
bgp.l3vpn.0          
                      52         52          0          0          0          0
bgp.l3vpn-inet6.0    
                       4          4          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.1              65020         40         57       0       0           3 Establ
  bgp.rtarget.0: 5/5/5/0
  inet.0: 25/85/85/0
  inet6.0: 3/18/18/0
  bgp.l3vpn.0: 7/7/7/0
10.0.0.2              65020         40         78       0       0           3 Establ
  bgp.rtarget.0: 1/6/6/0
  inet.0: 1/85/85/0
  inet6.0: 1/18/18/0
  bgp.l3vpn.0: 7/7/7/0
10.0.0.3              65020         48         71       0       0           3 Establ
  bgp.rtarget.0: 0/1/1/0
  inet.0: 72/137/137/0
  inet6.0: 19/23/23/0
  bgp.l3vpn.0: 10/10/10/0
  bgp.l3vpn-inet6.0: 0/0/0/0
10.0.0.4              65020         17         86       0       0           3 Establ
  bgp.rtarget.0: 3/8/8/0
  inet.0: 1/15/15/0
  inet6.0: 0/0/0/0
  bgp.l3vpn.0: 12/12/12/0
10.0.0.5              65020         14         78       0       0           3 Establ
  bgp.rtarget.0: 2/5/5/0
  inet.0: 8/20/20/0
  inet6.0: 1/1/1/0                      
  bgp.l3vpn.0: 6/6/6/0
10.0.0.6              65020         13         61       0       0           3 Establ
  bgp.rtarget.0: 0/1/1/0
  inet.0: 15/15/15/0
  inet6.0: 0/0/0/0
  bgp.l3vpn.0: 3/3/3/0
10.0.0.7              65020         16         63       0       0           3 Establ
  bgp.rtarget.0: 3/4/4/0
  inet.0: 0/1/1/0
  inet6.0: 0/0/0/0
  bgp.l3vpn.0: 3/3/3/0
10.0.0.8              65020         14         68       0       0           3 Establ
  bgp.rtarget.0: 1/2/2/0
  inet.0: 31/31/31/0
  inet6.0: 1/1/1/0
  bgp.l3vpn.0: 4/4/4/0
  bgp.l3vpn-inet6.0: 0/0/0/0
```
As you can see, only R3 and R8 will speak family inet6-vpn. Ok, I'll say the truth, R3 and R8 already have the VRF configured if they don't have the family would not be shown here. But I omitted the number of routers to make this pleasant for me, hahah. 

The interfaces already are configured and the customer doesn't have any particular requirement, so this service will be simple. With this clarified, let's go the configuration: 

First, let's define the routing-instance and configure the BGP session with the customer:
```
set routing-instances VRF-C3 instance-type vrf
set routing-instances VRF-C3 protocols bgp group eBGP-AS64703 type external
set routing-instances VRF-C3 protocols bgp group eBGP-AS64703 description eBGP-AS64703
set routing-instances VRF-C3 protocols bgp group eBGP-AS64703 peer-as 64703
set routing-instances VRF-C3 protocols bgp group eBGP-AS64703 as-override
set routing-instances VRF-C3 protocols bgp group eBGP-AS64703 neighbor fc09:c0:ffee:3:8::2 family inet6 unicast
set routing-instances VRF-C3 description VRF-C3
set routing-instances VRF-C3 interface ge-0/0/8.300
set routing-instances VRF-C3 interface lo0.2
set routing-instances VRF-C3 route-distinguisher 10.0.0.8:300
set routing-instances VRF-C3 vrf-target target:65020:300
set routing-instances VRF-C3 vrf-table-label
```
Again, we are using the as-override to avoid routes rejected by AS loop. If the 6VPE were equal inet-vpn, with this configuratio the customer could have communication between the sites, but we need a specific knob to 6VPE work. 

We need to add the ipv6-tunneling in the MPLS, we already made this before, to use 6PE in our backbone. But I'll show you here, to remember: 
```
set protocols mpls ipv6-tunneling
```
Now, we can apply the same logic to the R3. 

Let's check our RIB:
```
root@R8> show route table VRF-C3.inet6.0    

VRF-C3.inet6.0: 10 destinations, 11 routes (10 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

fc09::3:1/128      *[BGP/170] 00:00:02, localpref 100
                      AS path: 64703 I, validation-state: unverified
                    >  to fc09:c0:ffee:3:8::2 via ge-0/0/8.300
fc09::3:2/128      *[BGP/170] 00:00:33, localpref 100, from 10.0.0.0
                      AS path: 64703 I, validation-state: unverified
                    >  to 10.200.0.4 via ge-0/0/3.0, label-switched-path R8-R3-A
                       to 10.200.0.18 via ge-0/0/2.0, label-switched-path R8-R3-B
                       to 10.200.0.22 via ge-0/0/4.0, label-switched-path Bypass->10.200.0.4->10.200.0.1
fc09:c0:ffee:3:3::/126
                   *[BGP/170] 00:11:31, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.4 via ge-0/0/3.0, label-switched-path R8-R3-A
                       to 10.200.0.18 via ge-0/0/2.0, label-switched-path R8-R3-B
                       to 10.200.0.22 via ge-0/0/4.0, label-switched-path Bypass->10.200.0.4->10.200.0.1
fc09:c0:ffee:3:3::254/128
                   *[BGP/170] 00:11:31, localpref 100, from 10.0.0.0
                      AS path: I, validation-state: unverified
                    >  to 10.200.0.4 via ge-0/0/3.0, label-switched-path R8-R3-A
                       to 10.200.0.18 via ge-0/0/2.0, label-switched-path R8-R3-B
                       to 10.200.0.22 via ge-0/0/4.0, label-switched-path Bypass->10.200.0.4->10.200.0.1
fc09:c0:ffee:3:8::/126
                   *[Direct/0] 2w2d 23:14:20
                    >  via ge-0/0/8.300
                    [BGP/170] 00:00:02, localpref 100
                      AS path: 64703 I, validation-state: unverified
                    >  to fc09:c0:ffee:3:8::2 via ge-0/0/8.300
fc09:c0:ffee:3:8::1/128
                   *[Local/0] 2w2d 23:14:20
                       Local via ge-0/0/8.300
fc09:c0:ffee:3:8::254/128
                   *[Direct/0] 2w2d 23:15:21
                    >  via lo0.2
fe80::521e:d00f:fc00:1000/128
                   *[Direct/0] 2w2d 23:15:21
                    >  via lo0.2
fe80::528b:d801:2c00:120a/128
                   *[Local/0] 2w2d 23:14:20
                       Local via ge-0/0/8.300
ff02::2/128        *[INET6/0] 2w2d 23:15:21
                       MultiRecv
```
Now, let's test the connectivity between the sites!!!
```
[admin@CE3-1] > tool/traceroute src-address=fc09::3:1 fc09::3:2
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS              LOSS  SENT  LAST    AVG   BEST  WORST  STD-DEV
1  fc09:c0:ffee:3:8::1  0%       3  3ms     3.7   3     4.4    0.6    
2  fc09:c0:ffee:3:3::1  0%       3  59.2ms  61.1  52.3  71.9   8.1    
3  fc09::3:2            0%       3  7.7ms   9     6.4   12.8   2.8
```
And everything is perfect!!! 

I know, this time the text is shameful, so short... I'm sorry, the next article will be MVPN, write about multicast give me goosebumps, I HATE MULTICAST BRO! HELP ME! 

See you soon.




Yes, I'm sad just remembering multicast...

Bye.

Bye...
