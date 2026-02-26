# iBGP Peering Configuration

Hello guys! Today we have a massive mission to accomplish: configuring our BGP Free Core and 6PE structure.

Before we start, I need to make a quick disclaimer. For this configuration to be fully verifiable in the data plane, it must be deployed on top of a working MPLS foundation. The study guide I am following doesn't present the topics in this exact logical sequence. Therefore, my connectivity tests at the end of this article will be successful because I have already provisioned the underlying MPLS/SR-MPLS layer in my lab. Keep that in mind!

Our topology remains the same as in the previous articles:
<img width="1145" height="744" alt="image" src="https://github.com/user-attachments/assets/9cbe9972-0a7c-45d9-98b6-de1fcfbe4e3a" />


We have a few tasks to accomplish to make this network truly elegant:

* We need to integrate the Route Reflector into our IGP. They will connect to R1 and R2.

* The RR will be out-of-path, meaning they will only handle the Control Plane and will not run MPLS/SR-MPLS or forward transit traffic.

* All iBGP sessions must be established using TCP-AO authentication and BFD, and we must log state changes using the log-updown knob.

* We will deploy two RR clusters: East and West.

* R1, R6, R7, and R8 will be RR Clients in the West cluster.

* R2, R3, R4, and R5 will be RR Clients in the East cluster.

* All PE routers will apply the classic next-hop self policy for exported routes.

With our constraints defined, let's set up the RRs! The RRs are pre-configured with baseline settings, just like the other routers, but without MPLS. The RR loopback address is 10.0.0.0.

First, we need to configure the backbone interfaces on R1 and R2 to bring the RR into our IS-IS routing domain:
R1:
```
set interfaces ge-0/0/4 apply-groups isis-if
set interfaces ge-0/0/4 description to-RR/ge-0/0/0
set interfaces ge-0/0/4 unit 0 family inet address 10.200.0.24/31
```
R2:
```
set interfaces ge-0/0/4 apply-groups isis-if
set interfaces ge-0/0/4 description to-RR/ge-0/0/1
set interfaces ge-0/0/4 unit 0 family inet address 10.200.0.26/31
```

Let's verify the IS-IS adjacency to confirm the RR is reachable in the IGP:
```
root@R1> show isis adjacency RR detail 
RR
  Interface: ge-0/0/4.0, Level: 1, State: Up, Expires in 24 secs
  Priority: 0, Up/Down transitions: 1, Last transition: 1w2d 19:38:25 ago
  Circuit type: 1, Speaks: IP, IPv6
  Topologies: Unicast
  Restart capable: Yes, Adjacency advertisement: Advertise
  IP addresses: 10.200.0.25

root@R2> show isis adjacency RR detail 
RR
  Interface: ge-0/0/4.0, Level: 1, State: Up, Expires in 26 secs
  Priority: 0, Up/Down transitions: 1, Last transition: 1w2d 19:37:51 ago
  Circuit type: 1, Speaks: IP, IPv6
  Topologies: Unicast
  Restart capable: Yes, Adjacency advertisement: Advertise
  IP addresses: 10.200.0.27
```

Now that we have access to the RR, we can define the BGP groups with different cluster-id values for the East and West clusters.

Pro Tip: Because our RR is out-of-path and doesn't run MPLS, it doesn't need to install BGP routes into the hardware Forwarding Information Base (FIB), nor does it need to resolve next-hops using the inet.3 routing table. By adding the no-install and nexthop-resolution no-resolution knobs, we optimize the RR's resources. In my experience, this approach is much cleaner than dealing with complex rib-groups or resolution-ribs.
```
set protocols bgp group iBGP-AS65020-West type internal
set protocols bgp group iBGP-AS65020-West description iBGP-AS65020-West
set protocols bgp group iBGP-AS65020-West local-address 10.0.0.0
set protocols bgp group iBGP-AS65020-West log-updown
set protocols bgp group iBGP-AS65020-West family inet unicast nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-West family inet unicast no-install
set protocols bgp group iBGP-AS65020-East family inet6 labeled-unicast nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-East family inet6 labeled-unicast no-install
set protocols bgp group iBGP-AS65020-West cluster 0.0.0.1
set protocols bgp group iBGP-AS65020-West neighbor 10.0.0.1
set protocols bgp group iBGP-AS65020-West neighbor 10.0.0.6
set protocols bgp group iBGP-AS65020-West neighbor 10.0.0.7
set protocols bgp group iBGP-AS65020-West neighbor 10.0.0.8

set protocols bgp group iBGP-AS65020-East type internal
set protocols bgp group iBGP-AS65020-East description iBGP-AS65020-East
set protocols bgp group iBGP-AS65020-East local-address 10.0.0.0
set protocols bgp group iBGP-AS65020-East log-updown
set protocols bgp group iBGP-AS65020-East family inet unicast nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-East family inet unicast no-install
set protocols bgp group iBGP-AS65020-East family inet6 labeled-unicast nexthop-resolution no-resolution
set protocols bgp group iBGP-AS65020-East family inet6 labeled-unicast no-install
set protocols bgp group iBGP-AS65020-East cluster 0.0.0.2
set protocols bgp group iBGP-AS65020-East neighbor 10.0.0.2
set protocols bgp group iBGP-AS65020-East neighbor 10.0.0.3
set protocols bgp group iBGP-AS65020-East neighbor 10.0.0.4
set protocols bgp group iBGP-AS65020-East neighbor 10.0.0.5
```

To improve failure detection, we apply a BFD interval of 3000ms, matching our IGP baseline:
```
set protocols bgp group iBGP-AS65020-East bfd-liveness-detection minimum-interval 3000
set protocols bgp group iBGP-AS65020-West bfd-liveness-detection minimum-interval 3000
```

Next, we replace legacy MD5 with TCP-AO to secure the TCP sessions established between our routers. We define the authentication key-chain, specify a start-time for key rotation, and apply it to the BGP groups.
```
set security authentication-key-chains key-chain l4b key 0 secret l4b
set security authentication-key-chains key-chain l4b key 0 start-time "2026-25-2.14:25:00 -0300"
set security authentication-key-chains key-chain l4b key 0 algorithm ao
set security authentication-key-chains key-chain l4b key 0 ao-attribute send-id 1
set security authentication-key-chains key-chain l4b key 0 ao-attribute recv-id 2
set security authentication-key-chains key-chain l4b key 0 ao-attribute tcp-ao-option enabled
set security authentication-key-chains key-chain l4b key 0 ao-attribute cryptographic-algorithm aes-128-cmac-96

set protocols bgp group iBGP-AS65020-East authentication-algorithm ao
set protocols bgp group iBGP-AS65020-East authentication-key-chain l4b
set protocols bgp group iBGP-AS65020-West authentication-algorithm ao
set protocols bgp group iBGP-AS65020-West authentication-key-chain l4b
```
Note: The send-id on the RR must match the recv-id on the client, and vice versa. E.g., if the RR sends with ID 1 and receives with ID 2, the client will send with ID 2 and receive with ID 1.

With the RR fully configured, we move on to the RR Clients. I will use R1 as the baseline example; the other routers follow an identical structure, differing only by their local-address.

We define an import policy to reject routes tagged with the Blackhole community, and an export policy to enforce next-hop self.
```
set policy-options policy-statement next-hop-self then next-hop self
set policy-options policy-statement import-rr term BH from community Blackhole
set policy-options policy-statement import-rr term BH then next-hop discard
set policy-options policy-statement import-rr term BH then accept
set policy-options policy-statement import-rr then accept
```

Next, we configure the TCP-AO key-chain on the client side, ensuring we invert the send-id and recv-id values:
```
et security authentication-key-chains key-chain l4b key 0 secret l4b
set security authentication-key-chains key-chain l4b key 0 start-time "2026-25-2.14:25:00 -0300"
set security authentication-key-chains key-chain l4b key 0 algorithm ao
set security authentication-key-chains key-chain l4b key 0 ao-attribute send-id 2
set security authentication-key-chains key-chain l4b key 0 ao-attribute recv-id 1
set security authentication-key-chains key-chain l4b key 0 ao-attribute tcp-ao-option enabled
set security authentication-key-chains key-chain l4b key 0 ao-attribute cryptographic-algorithm aes-128-cmac-96
```
Note that the values are inverted of the values of the RR! 

Finally, we apply the BGP session parameters. Important: To make the 6PE functionality work seamlessly, we must configure family inet6 labeled-unicast explicit-null and ipv6-tunneling in MPLS (as I said, I finished the MPLS configuration earlier). 
```
set protocols bgp group iBGP-AS65020-West type internal
set protocols bgp group iBGP-AS65020-West description iBGP-AS65020-West
set protocols bgp group iBGP-AS65020-West local-address 10.0.0.1
set protocols bgp group iBGP-AS65020-West import import-rr
set protocols bgp group iBGP-AS65020-West family inet unicast
set protocols bgp group iBGP-AS65020-West family inet6 labeled-unicast explicit-null
set protocols bgp group iBGP-AS65020-West authentication-algorithm ao
set protocols bgp group iBGP-AS65020-West authentication-key-chain l4b
set protocols bgp group iBGP-AS65020-West export next-hop-self
set protocols bgp group iBGP-AS65020-West export Saida-RR
set protocols bgp group iBGP-AS65020-West bfd-liveness-detection minimum-interval 3000
set protocols bgp group iBGP-AS65020-West neighbor 10.0.0.0
```

Let's check the RR to ensure all sessions are established:
```
root@RR> show bgp summary    
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 2 Peers: 8 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0               
                     321        147          0          0          0          0
inet6.0              
                      42         24          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.1              65020         31         43       0       0          23 Establ
  inet.0: 28/88/88/0
  inet6.0: 4/18/18/0                    
10.0.0.2              65020         31         35       0       0          23 Establ
  inet.0: 1/88/88/0
  inet6.0: 14/18/18/0
10.0.0.3              65020         13         46       0       0          23 Establ
  inet.0: 63/65/65/0
  inet6.0: 4/4/4/0
10.0.0.4              65020          6         52       0       0          23 Establ
  inet.0: 1/13/13/0
  inet6.0: 0/0/0/0
10.0.0.5              65020          8         50       0       0          22 Establ
  inet.0: 8/20/20/0
  inet6.0: 1/1/1/0
10.0.0.6              65020          8         49       0       0          23 Establ
  inet.0: 15/15/15/0
  inet6.0: 0/0/0/0
10.0.0.7              65020         11         39       0       0          22 Establ
  inet.0: 0/1/1/0                       
  inet6.0: 0/0/0/0
10.0.0.8              65020          6         37       0       0          21 Establ
  inet.0: 31/31/31/0
  inet6.0: 1/1/1/0
```
The routes are being received and installed in the RIB, but thanks to our no-install knob, they remain out of the FIB!

Now, let's verify if the RR Clients are receiving the reflected routes. Here is the view from R5:
```
root@R5> show route receive-protocol bgp 10.0.0.0 

inet.0: 227 destinations, 435 routes (222 active, 0 holddown, 199 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  0.0.0.0/0               10.0.0.1                     100        I
* 1.1.1.0/24              10.0.0.1                     100        1620 111 I
* 8.8.8.0/24              10.0.0.1                     100        1620 888 I
* 162.0.1.0/24            10.0.0.1                     100        1620 I
* 162.0.2.0/24            10.0.0.1                     100        1620 I
* 162.0.3.0/24            10.0.0.1                     100        1620 I
...
inet6.0: 40 destinations, 68 routes (39 active, 0 holddown, 9 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 2804:db8:111::/48       ::ffff:10.0.0.3              100        65503 111 I
* 2804:db8:888::/48       ::ffff:10.0.0.3              100        65503 888 I
* 2804:2001::/48          ::ffff:10.0.0.8              200        65501 I
* 2804:2002::/48          ::ffff:10.0.0.3              100        65502 I
* 2804:2003::/48          ::ffff:10.0.0.3              100        65503 I
```
Everything looks perfect.


The moment of truth! Let's verify end-to-end connectivity by performing traceroutes from Customer 5 (CE5-3) across our newly established BGP Free Core towards all other peerings.
CE5-3 -> IX-1 and IX-2:
```
[admin@CE5-3] > tool traceroute src-address=201.5.0.3 address=162.0.1.1
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS         LOSS  SENT  LAST     AVG   BEST  WORST  STD-DEV
1  172.16.5.1      0%       4  149.7ms  56.3  3.9   149.7  66.2   
2  10.200.0.2      0%       3  31ms     36.6  31    44.2   5.6    
3  192.168.12.253  0%       3  29.2ms   95.3  16.7  239.9  102.4  
4  162.0.1.1       0%       3  19.2ms   26.9  19.2  32.3   5.6    

[admin@CE5-3] > tool traceroute src-address=201.5.0.3 address=162.0.1.2
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS     LOSS  SENT  LAST    AVG    BEST  WORST  STD-DEV
1  172.16.5.1  0%       4  6.2ms   28.7   6.2   78     28.9   
2  10.200.0.2  0%       4  28ms    189.6  28    522.7  197.2  
3  162.0.1.2   0%       4  15.9ms  33.1   10.4  80.3   27.8   
```
CE5-3 -> P2-1:
```
[admin@CE5-3] > tool traceroute src-address=201.5.0.3 address=200.2.0.1
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS      LOSS  SENT  LAST    AVG   BEST  WORST  STD-DEV
1  172.16.5.1   0%       3  1.8ms   35.8  1.8   98.3   44.3   
2  10.200.0.10  0%       3  46.8ms  30.9  14.9  46.8   16     
3  200.2.0.1    0%       2  14.6ms  31.9  14.6  49.2   17.3

[admin@CE5-3] > tool traceroute src-address=2804:2015::3 address=2804:2002::
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS            LOSS  SENT  LAST    AVG   BEST  WORST  STD-DEV
1  fc09:c0:ffee:5::1  0%       3  13.6ms  8.4   3.2   13.6   5.2    
2  fd10:faca:f0fa::3  0%       2  57.6ms  36.4  15.2  57.6   21.2   
3  2804:2002::        0%       2  22.4ms  16.2  9.9   22.4   6.3     
```
CE5-3 -> P3-1
```
[admin@CE5-3] > tool traceroute src-address=201.5.0.3 address=200.3.0.1
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS      LOSS  SENT  LAST    AVG   BEST  WORST  STD-DEV
1  172.16.5.1   0%       3  48.3ms  21.6  8.1   48.3   18.9   
2  10.200.0.10  0%       3  25.3ms  21.4  12.4  26.6   6.4    
3  200.3.0.1    0%       3  14.9ms  12    9.8   14.9   2.1    

[admin@CE5-3] > tool traceroute src-address=2804:2015::3 address=2804:2003::
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS            LOSS  SENT  LAST    AVG   BEST  WORST  STD-DEV
1  fc09:c0:ffee:5::1  0%       3  5.6ms   14.7  5.6   27.4   9.3    
2  fd10:faca:f0fa::3  0%       3  52.5ms  45.7  36.4  52.5   6.8    
3  2804:2003::        0%       3  30.8ms  32.1  30.6  34.9   2      
```
CE5-3 -> P1-1 and P1-2:
```
[admin@CE5-3] > tool traceroute src-address=201.5.0.3 address=200.1.0.1
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS      LOSS  SENT  LAST    AVG    BEST  WORST  STD-DEV
1  172.16.5.1   0%       4  29.9ms  82.3   2.4   230.4  88.5   
2  10.200.0.19  0%       4  44.5ms  210.7  23    669.9  266.9  
3  200.1.0.1    0%       4  6.7ms   12.8   4.1   32.1   11.3   

[admin@CE5-3] > tool traceroute src-address=2804:2015::3 address=2804:2001::1
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS            LOSS  SENT  LAST    AVG    BEST  WORST  STD-DEV
1  fc09:c0:ffee:5::1  0%       6  7.6ms   75.8   3     406.7  148.1  
2  fd10:faca:f0fa::8  0%       6  34.3ms  182.3  9.4   895.2  319.8  
3  2804:2001::1       0%       6  65.5ms  24.5   9.5   65.5   19.4

[admin@CE5-3] > tool traceroute src-address=201.5.0.3 address=200.1.0.2
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS      LOSS  SENT  LAST    AVG    BEST  WORST  STD-DEV
1  172.16.5.1   0%       4  4.9ms   145.2  4.9   551.3  234.5  
2  10.200.0.19  0%       4  20.7ms  29.7   5.1   84     31.9   
3  172.16.8.2   0%       4  5.2ms   8.2    5.2   14.4   3.7    
4  200.1.0.2    0%       4  12.6ms  12.7   7.4   18     3.7

[admin@CE5-3] > tool traceroute src-address=2804:2015::3 address=2804:2001::2
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS            LOSS  SENT  LAST    AVG    BEST  WORST  STD-DEV
1  fc09:c0:ffee:5::1  0%       3  16.7ms  58.6   12.3  146.7  62.3   
2  fd10:faca:f0fa::8  0%       3  29ms    135.9  29    346.6  149    
3  fc09:c0:ffee:8::2  0%       3  13.9ms  13.5   9.5   17     3.1    
4  2804:2001::2       0%       3  15.9ms  11.1   7.2   15.9   3.6     
```
CE5-3 -> C2-1:                           
```
[admin@CE5-3] > tool traceroute src-address=201.5.0.3 address=201.2.1.1
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS      LOSS  SENT  LAST    AVG   BEST  WORST  STD-DEV
1  172.16.5.1   0%       4  2.8ms   10.5  1.9   26.7   11.5   
2  10.200.0.17  0%       3  4.4ms   41.3  4.4   88     34.8   
3  201.2.1.1    0%       3  10.9ms  13.3  6.2   22.9   7      
```
CE5-3 -> C1-1:                           
```
[admin@CE5-3] > tool traceroute src-address=201.5.0.3 address=201.1.0.1
Columns: ADDRESS, LOSS, SENT, LAST, AVG, BEST, WORST, STD-DEV
#  ADDRESS      LOSS  SENT  LAST   AVG   BEST  WORST  STD-DEV
1  172.16.5.1   0%       4  8.3ms  7.7   2     16.1   5.3    
2  10.200.0.17  0%       4  4.3ms  98.5  4.3   374.4  159.3  
3  201.1.0.1    0%       4  3ms    5.6   3     8.3    2.2    

[admin@CE5-3] > 
```
We have full connectivity! Our core is routing efficiently, and our network is fully primed to deliver IP services.

See you in the next chapter, where we will dive into the underlying MPLS/SR-MPLS configurations that made this transit possible!

