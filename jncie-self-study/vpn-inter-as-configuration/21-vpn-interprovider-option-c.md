# VPN Inter-Provider Option C Configuration

Hello friends, this time is the last VPN that we'll configure in our JNCIE-SP journey. This is sad, but in the same time is happy. We've explored so many features in this journey. 

This is topology that we'll work today:
<img width="1330" height="913" alt="image" src="https://github.com/user-attachments/assets/55aa7b65-0984-4cff-a42e-ae5fd49cf7c4" />

Our goal is deliver the VPLS service to Customer 5, connected at P3. Following the same lore of the previous article, we acquired P3, and now we can interconnect the services of the common customers. 

This time, we'll use the VPN Inter-AS Option C, in another words, we'll use the BGP-LU to advertise our loopbacks with labels to P3, and the P3 will advertise his own loopback with labels to us. And now, we'll interconnect the P3 with our network. 

You got the logic? When P3 wants to forward the traffic to R1 for example, it will use the label that we advertised trough BGP, then R3 will receive the packet with this label and swap to another label and forward the traffic to our network. And the reverse happens too, when R1 wants to forward traffic to P3, it pushes the label receive on the BGP-LU route, and push another one to R3, with the PHP on the network, R3 will receive the packet with BGP-LU label only, and swaps this label then forward to P3. 

Let's configure this now, first, let's enable the family inet labeled-unicast on our BGP mesh:
RR:
```
set protocols bgp group iBGP-AS65020-West family inet labeled-unicast rib inet.3
set protocols bgp group iBGP-AS65020-East family inet labeled-unicast rib inet.3
```
PEs:
```
set protocols bgp group iBGP-AS65020-East family inet labeled-unicast rib inet.3
```

Now that we have the family connected on our backbone, we can establish a BGP-LU peering with P3. Now, we need to advertise the loopback routes of our network to P3, so we just need to create another term to do this. 
```
set protocols bgp group eBGP-AS65503-Provider3 family inet labeled-unicast rib inet.3

set policy-options policy-statement Saida-P3 term from-igp from route-filter 10.0.0.0/24 upto /32
set policy-options policy-statement Saida-P3 term from-igp then accept
```
With this, we'll export all these routes present on inet.3:
```
root@R3> show route table inet.3 10.0.0.0/24 active-path

inet.3: 72 destinations, 133 routes (72 active, 0 holddown, 40 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/32        *[IS-IS/15] 5w0d 03:11:50, metric 16777224
                    >  to 10.200.0.6 via ge-0/0/3.0
10.0.0.1/32        *[LDP/9] 3w0d 03:11:02, metric 15
                    >  to 10.200.0.6 via ge-0/0/3.0, Push 33
                       to 10.200.0.11 via ge-0/0/4.0, Push 33
10.0.0.2/32        *[LDP/9] 3w0d 03:10:55, metric 10
                    >  to 10.200.0.6 via ge-0/0/3.0, Push 0
                       to 10.200.0.11 via ge-0/0/4.0, Push 35
10.0.0.3/32        *[Direct/0] 5w5d 23:51:38
                    >  via lo0.0
10.0.0.4/32        *[RSVP/7/1] 3w0d 03:10:06, metric 10
                    >  to 10.200.0.13 via ge-0/0/2.0, label-switched-path to-R4
10.0.0.5/32        *[LDP/9] 3w0d 03:11:02, metric 16777219
                    >  to 10.200.0.11 via ge-0/0/4.0, label-switched-path R3-R6-A
                       to 10.200.0.6 via ge-0/0/3.0, label-switched-path Bypass->10.200.0.11
10.0.0.6/32        *[RSVP/7/1] 5w0d 03:11:33, metric 16777214
                    >  to 10.200.0.11 via ge-0/0/4.0, label-switched-path R3-R6-A
                       to 10.200.0.6 via ge-0/0/3.0, label-switched-path Bypass->10.200.0.11
10.0.0.7/32        *[LDP/9] 3w0d 03:10:55, metric 16777224
                    >  to 10.200.0.11 via ge-0/0/4.0, label-switched-path R3-R6-A
                       to 10.200.0.6 via ge-0/0/3.0, label-switched-path Bypass->10.200.0.11
10.0.0.8/32        *[RSVP/7/1] 3w0d 03:10:45, metric 16777229
                    >  to 10.200.0.13 via ge-0/0/2.0, label-switched-path R3-R8-B
                       to 10.200.0.6 via ge-0/0/3.0, label-switched-path R3-R8-A
                       to 10.200.0.11 via ge-0/0/4.0, label-switched-path Bypass->10.200.0.6->10.200.0.0
```
Let's check the advertisements now: 
```
root@R3> show route advertising-protocol bgp 172.16.3.6 table inet.3

inet.3: 73 destinations, 134 routes (73 active, 0 holddown, 40 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.0.0.0/32             Self                 16777224           I
* 10.0.0.1/32             Self                 15                 I
* 10.0.0.2/32             Self                 10                 I
* 10.0.0.3/32             Self                                    I
* 10.0.0.4/32             Self                 10                 I
* 10.0.0.5/32             Self                 16777219           I
* 10.0.0.6/32             Self                 16777214           I
* 10.0.0.7/32             Self                 16777224           I
* 10.0.0.8/32             Self                 16777229           I

root@R3> show route receive-protocol bgp 172.16.3.6 table inet.3

inet.3: 73 destinations, 134 routes (73 active, 0 holddown, 40 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.0.1.3/32             172.16.3.6                              65503 I
```
Everything is good until here. Now, we need to configure the service. 

P3 will establish a session with our RR, and advertises the l2vpn routes to interconnect the sites. Let's configure this sessions and check what is the RT of the route advertised. 
```
set protocols bgp group eBGP-AS65503-LU type external
set protocols bgp group eBGP-AS65503-LU description eBGP-AS65503-LU
set protocols bgp group eBGP-AS65503-LU multihop ttl 24
set protocols bgp group eBGP-AS65503-LU local-address 10.0.0.0
set protocols bgp group eBGP-AS65503-LU peer-as 65503
set protocols bgp group eBGP-AS65503-LU neighbor 10.0.1.3 family l2vpn signaling
```
Basically, this is a multihop eBGP session, that speaks l2vpn family. 
