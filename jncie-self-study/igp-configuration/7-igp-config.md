# IGP Configuration

Hello everyone, 

Today is the big day. 

Today, we'll configure our IGP. Our goals is:
* Have connectivity with our new DCs, DC1 and DC2. 
* Have full conectivity IPv4 and IPv6 between our routers. 
* Establish full connectivity between our DCs and IGP.

Basically this, with some details also. 

In our IGP, we"ll run IS-IS. 

For practice purposes, we'll configure the connection with the DC2 using BGP, and with the DC3 we'll use OSPFv3. 
For the full connectivity between the DCs and our IGP, we need to leak some routes from other protocols in the ISIS and vice-versa. 

Ok, with this mindset, here we go:
This is our topology today:
<img width="1230" height="790" alt="image" src="https://github.com/user-attachments/assets/181ac158-9831-47d6-b873-e54394b24f3b" />

First things first, let's configure the new interfaces to our DCs, we'll follow the table below to configure the interfaces:
| Router | Interface | IP address | IPv6 address |
| - | - | - | - |
| R4 | ge-0/0/1.0 | 172.30.120.1/30 | |
| R4 | ge-0/0/5.0 | 172.30.130.1/30 | link-local |
| R5 | ge-0/0/5.0 | 172.30.120.5/30 | |
| R5 | ge-0/0/6.0 | 172.30.130.5/30 | link-local |

The configuration is:

R4:
```
set interfaces ge-0/0/1 description to-DC2/ge-0/0/0
set interfaces ge-0/0/1 unit 0 family inet address 172.30.120.1/30
set interfaces ge-0/0/5 description to-DC3/ge-0/0/0
set interfaces ge-0/0/5 unit 0 family inet address 172.30.130.1/30
set interfaces ge-0/0/5 unit 0 family inet6
```
R5:
```
set interfaces ge-0/0/5 description to-DC2/ge-0/0/1
set interfaces ge-0/0/5 unit 0 family inet address 172.30.120.5/30
set interfaces ge-0/0/6 description to-DC3/ge-0/0/1
set interfaces ge-0/0/6 unit 0 family inet address 172.30.130.5/30
set interfaces ge-0/0/6 unit 0 family inet6
```
You want to check the connectivity, right? To kill your doubt, I know, I know...
```
root@R4> ping 172.30.120.2 count 1
PING 172.30.120.2 (172.30.120.2): 56 data bytes
64 bytes from 172.30.120.2: icmp_seq=0 ttl=64 time=3.394 ms

--- 172.30.120.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 3.394/3.394/3.394/0.000 ms

root@R4> ping 172.30.130.2 count 1
PING 172.30.130.2 (172.30.130.2): 56 data bytes
64 bytes from 172.30.130.2: icmp_seq=0 ttl=64 time=2.431 ms

--- 172.30.130.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.431/2.431/2.431/0.000 ms

root@R5> ping 172.30.120.6 count 1
PING 172.30.120.6 (172.30.120.6): 56 data bytes
64 bytes from 172.30.120.6: icmp_seq=0 ttl=64 time=2.360 ms

--- 172.30.120.6 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.360/2.360/2.360/0.000 ms

root@R5> ping 172.30.130.6 count 1
PING 172.30.130.6 (172.30.130.6): 56 data bytes
64 bytes from 172.30.130.6: icmp_seq=0 ttl=64 time=4.762 ms

--- 172.30.130.6 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 4.762/4.762/4.762/0.000 ms
```

Ok, let's continue our work. Now, we'll start the IS-IS configuration

For this, we need to configure the Network Entity address for the IS-IS run, and we need to configure the protocol and family iso in the backbone interfaces. 

For control, let's make a table of the NET address first:
| Router | NET address |
| - | - |
| R1 | 49.0001.0100.0000.0001.00 | 
| R2 | 49.0001.0100.0000.0002.00 |
| R3 | 49.0001.0100.0000.0003.00 |
| R4 | 49.0001.0100.0000.0004.00 |
| R5 | 49.0001.0100.0000.0005.00 |
| R6 | 49.0001.0100.0000.0006.00 |
| R7 | 49.0001.0100.0000.0007.00 |
| R8 | 49.0001.0100.0000.0008.00 |

If you remember the loopback address of our routers, you can understand the NET address. This is good practice to do, use the loopback address as router id, and as a base for the NET address. 

For example, we have the 10.0.0.1 as loopback address of R1.

So, we'll enjoy the NET structure to introduce this address as System ID on NET address.

The structure of NET address is {AFI}.{AREA}.{SYSTEM-ID}.{NSEL}
So, in our NET addresses we have:
* AFI - 49
* AREA - 0001
* SYSTEM-ID - IP Address = 010.000.000.001 / SYS-ID = 0100.0000.0001
* NSEL - 0

Resulting in a NET address = 49.0001.0100.0000.0001.00

Now it's clear, right? So, let's make the configuration of the NET address and configure the router ID of our routers explicitly. I'll show the R1 configuration only for summarization purposes. 
```
set interfaces lo0 unit 0 family iso address 49.0001.0100.0000.0001.00
set routing-options router-id 10.0.0.1
```
Now, for the backbone interface, let's use the groups in Junos. 
With this, we can apply this group and the interface will inherit the configuration of the group. 

Let's do it:
```
set groups isis-if interfaces <*> mtu 9216
set groups isis-if interfaces <*> unit <*> family inet mtu 9000
set groups isis-if interfaces <*> unit <*> family iso
set interfaces ge-0/0/2 apply-groups isis-if
set interfaces ge-0/0/3 apply-groups isis-if
set interfaces ae0 apply-groups isis-if
```
But, how can I be sure if the configuration is applied?
As I said, the interface will inherit the configuration, in Junos ou can view the inheritance of a configuration:
```
root@R1> show configuration interfaces ae0 | display inheritance
description to-R2/ae0;
##
## '9216' was inherited from group 'isis-if'
##
mtu 9216;
aggregated-ether-options {
    lacp {
        active;
    }
}
unit 0 {
    family inet {
        ##
        ## '9000' was inherited from group 'isis-if'
        ##
        mtu 9000;
        address 10.200.0.0/31;
    }
    ##
    ## 'iso' was inherited from group 'isis-if'
    ##
    family iso;
    family inet6;
}
```
So cool, right? You can do wonderful things and save some lines of your configuration with the groups! 


Now, we have to go for IS-IS configuration. 

We'll use only the level 1, and only one area. To have more metrics, we'll use wide metrics in our IS-IS. 

For simulate a good practice of security, we'll use authentication MD5 in the adjacencies and for better failure detection, we'll use BFD in all backbone interfaces with 2s of minimum-interval and with a multiplier value of 3.

Ok, in real-life looks like a big time of loss, but we are in a lab environment, don't worry. 

For optimize the costs of IGP, we'll use a reference bandwitdh of 10G, with this, our ae interface will have a lower cost than our single links. 

Because family iso is needed to run IS-IS on the interfaces, we have a facility in the configuration of the protocol. We can set the configuration for all interface in one time, because all the interfaces of the router considered by protocol IS-IS, are only the interface with the family iso configured. 
```
set protocols isis interface all point-to-point
set protocols isis interface all family inet bfd-liveness-detection minimum-interval 2000
set protocols isis interface all family inet bfd-liveness-detection multiplier 3
set protocols isis level 2 disable
set protocols isis level 1 authentication-key 1515-L4B
set protocols isis level 1 authentication-type md5
set protocols isis level 1 wide-metrics-only
set protocols isis reference-bandwidth 10g
```

With this, all interfaces with iso family are considered IS-IS interfaces:
```
root@R1> show isis interface
IS-IS interface database:
Interface             L CirID Level 1 DR        Level 2 DR        L1/L2 Metric
ae0.0                 1   0x1 Point to Point    Disabled                5/5
ge-0/0/2.0            1   0x1 Point to Point    Disabled               10/10
ge-0/0/3.0            1   0x1 Point to Point    Disabled               10/10
lo0.0                 1   0x1 Passive           Passive                 0/0
```
Look the lo0.0 interface, it is passive naturally, we don't need to set it passive. 

Ok, now we need to make this configuration in all routers of our network. I challenge you to make this in 5 minutes (if you are accompanying me in this lab, sure.). 

Ok, let's check our IS-IS database now:
```
root@R1> show isis database detail
IS-IS level 1 link-state database:
ISIS instance: Default, Routing Instance: master

R1.00-00 Sequence: 0x794, Checksum: 0xa895, Lifetime: 1084 secs
   IS neighbor: R2.00                         Metric:        5
   IS neighbor: R4.00                         Metric:       10
   IS neighbor: R8.00                         Metric:       10
   IP prefix: 10.0.0.1/32                     Metric:        0 Internal Up
   IP prefix: 10.200.0.0/31                   Metric:        5 Internal Up
   IP prefix: 10.200.0.2/31                   Metric:       10 Internal Up
   IP prefix: 10.200.0.4/31                   Metric:       10 Internal Up
   V6 prefix: fd10:faca:f0fa::1/128           Metric:        0 Internal Up

R2.00-00 Sequence: 0x79b, Checksum: 0xcd93, Lifetime: 1094 secs
   IS neighbor: R1.00                         Metric:        5
   IS neighbor: R3.00                         Metric:       10
   IS neighbor: R7.00                         Metric:       10
   IP prefix: 10.0.0.2/32                     Metric:        0 Internal Up
   IP prefix: 10.200.0.0/31                   Metric:        5 Internal Up
   IP prefix: 10.200.0.6/31                   Metric:       10 Internal Up
   IP prefix: 10.200.0.8/31                   Metric:       10 Internal Up
   V6 prefix: fd10:faca:f0fa::2/128           Metric:        0 Internal Up

R3.00-00 Sequence: 0x78b, Checksum: 0x1a22, Lifetime: 1124 secs
   IS neighbor: R2.00                         Metric:       10
   IS neighbor: R4.00                         Metric:       10
   IS neighbor: R6.00                         Metric:       10
   IP prefix: 10.0.0.3/32                     Metric:        0 Internal Up
   IP prefix: 10.200.0.6/31                   Metric:       10 Internal Up
   IP prefix: 10.200.0.10/31                  Metric:       10 Internal Up
   IP prefix: 10.200.0.12/31                  Metric:       10 Internal Up
   V6 prefix: fd10:faca:f0fa::3/128           Metric:        0 Internal Up

R4.00-00 Sequence: 0x863, Checksum: 0xa013, Lifetime: 1140 secs
   IS neighbor: R1.00                         Metric:       10
   IS neighbor: R3.00                         Metric:       10
   IS neighbor: R5.00                         Metric:       10
   IP prefix: 10.0.0.4/32                     Metric:        0 Internal Up
   IP prefix: 10.200.0.2/31                   Metric:       10 Internal Up
   IP prefix: 10.200.0.10/31                  Metric:       10 Internal Up
   IP prefix: 10.200.0.14/31                  Metric:       10 Internal Up
   V6 prefix: fd10:faca:f0fa::4/128           Metric:        0 Internal Up

R5.00-00 Sequence: 0x85f, Checksum: 0xd27c, Lifetime: 1150 secs
   IS neighbor: R4.00                         Metric:       10
   IS neighbor: R6.00                         Metric:        5
   IS neighbor: R8.00                         Metric:       10
   IP prefix: 10.0.0.5/32                     Metric:        0 Internal Up
   IP prefix: 10.200.0.14/31                  Metric:       10 Internal Up
   IP prefix: 10.200.0.16/31                  Metric:        5 Internal Up
   IP prefix: 10.200.0.18/31                  Metric:       10 Internal Up
   V6 prefix: fd10:faca:f0fa::5/128           Metric:        0 Internal Up

R6.00-00 Sequence: 0x772, Checksum: 0x99f6, Lifetime: 1162 secs
   IS neighbor: R3.00                         Metric:       10
   IS neighbor: R5.00                         Metric:        5
   IS neighbor: R7.00                         Metric:       10
   IP prefix: 10.0.0.6/32                     Metric:        0 Internal Up
   IP prefix: 10.200.0.12/31                  Metric:       10 Internal Up
   IP prefix: 10.200.0.16/31                  Metric:        5 Internal Up
   IP prefix: 10.200.0.20/31                  Metric:       10 Internal Up
   V6 prefix: fd10:faca:f0fa::6/128           Metric:        0 Internal Up

R7.00-00 Sequence: 0x772, Checksum: 0x7bbc, Lifetime: 1174 secs
   IS neighbor: R2.00                         Metric:       10
   IS neighbor: R6.00                         Metric:       10
   IS neighbor: R8.00                         Metric:       10
   IP prefix: 10.0.0.7/32                     Metric:        0 Internal Up
   IP prefix: 10.200.0.8/31                   Metric:       10 Internal Up
   IP prefix: 10.200.0.20/31                  Metric:       10 Internal Up
   IP prefix: 10.200.0.22/31                  Metric:       10 Internal Up
   V6 prefix: fd10:faca:f0fa::7/128           Metric:        0 Internal Up

R8.00-00 Sequence: 0x76e, Checksum: 0x2c9a, Lifetime: 1193 secs
   IS neighbor: R1.00                         Metric:       10
   IS neighbor: R5.00                         Metric:       10
   IS neighbor: R7.00                         Metric:       10
   IP prefix: 10.0.0.8/32                     Metric:        0 Internal Up
   IP prefix: 10.200.0.4/31                   Metric:       10 Internal Up
   IP prefix: 10.200.0.18/31                  Metric:       10 Internal Up
   IP prefix: 10.200.0.22/31                  Metric:       10 Internal Up
   V6 prefix: fd10:faca:f0fa::8/128           Metric:        0 Internal Up

IS-IS level 2 link-state database:
ISIS instance: Default, Routing Instance: master
```
Everything looks good, right? Are you sure if everything is ok? Are you sure?

<img width="220" height="234" alt="image" src="https://github.com/user-attachments/assets/a9d1899c-7dff-4741-922d-77b467bc7987" />

So, let's check the connectivity...
```
root@R1> show route 10.0.0.0/24

inet.0: 26 destinations, 26 routes (26 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.1/32        *[Direct/0] 2w3d 23:02:22
                    >  via lo0.0
10.0.0.2/32        *[IS-IS/15] 3d 05:53:13, metric 5
                    >  to 10.200.0.1 via ae0.0
10.0.0.3/32        *[IS-IS/15] 3d 05:53:13, metric 15
                    >  to 10.200.0.1 via ae0.0
10.0.0.4/32        *[IS-IS/15] 1w0d 00:18:21, metric 10
                    >  to 10.200.0.3 via ge-0/0/2.0
10.0.0.5/32        *[IS-IS/15] 1w0d 00:18:21, metric 20
                       to 10.200.0.3 via ge-0/0/2.0
                    >  to 10.200.0.5 via ge-0/0/3.0
10.0.0.6/32        *[IS-IS/15] 3d 05:53:13, metric 25
                       to 10.200.0.3 via ge-0/0/2.0
                       to 10.200.0.1 via ae0.0
                    >  to 10.200.0.5 via ge-0/0/3.0
10.0.0.7/32        *[IS-IS/15] 3d 05:53:13, metric 15
                    >  to 10.200.0.1 via ae0.0
10.0.0.8/32        *[IS-IS/15] 2w2d 23:13:21, metric 10
                    >  to 10.200.0.5 via ge-0/0/3.0

root@R1> ping count 1 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: icmp_seq=0 ttl=64 time=3.133 ms

--- 10.0.0.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 3.133/3.133/3.133/0.000 ms

root@R1> ping count 1 10.0.0.3
PING 10.0.0.3 (10.0.0.3): 56 data bytes
64 bytes from 10.0.0.3: icmp_seq=0 ttl=63 time=2.792 ms

--- 10.0.0.3 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.792/2.792/2.792/0.000 ms

root@R1> ping count 1 10.0.0.4
PING 10.0.0.4 (10.0.0.4): 56 data bytes
64 bytes from 10.0.0.4: icmp_seq=0 ttl=64 time=3.124 ms

--- 10.0.0.4 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 3.124/3.124/3.124/0.000 ms

root@R1> ping count 1 10.0.0.5
PING 10.0.0.5 (10.0.0.5): 56 data bytes
64 bytes from 10.0.0.5: icmp_seq=0 ttl=63 time=2.835 ms

--- 10.0.0.5 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.835/2.835/2.835/0.000 ms

root@R1> ping count 1 10.0.0.6
PING 10.0.0.6 (10.0.0.6): 56 data bytes
64 bytes from 10.0.0.6: icmp_seq=0 ttl=62 time=3.726 ms

--- 10.0.0.6 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 3.726/3.726/3.726/0.000 ms

root@R1> ping count 1 10.0.0.7
PING 10.0.0.7 (10.0.0.7): 56 data bytes
64 bytes from 10.0.0.7: icmp_seq=0 ttl=63 time=2.826 ms

--- 10.0.0.7 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.826/2.826/2.826/0.000 ms

root@R1> ping count 1 10.0.0.8
PING 10.0.0.8 (10.0.0.8): 56 data bytes
64 bytes from 10.0.0.8: icmp_seq=0 ttl=64 time=1.729 ms

--- 10.0.0.8 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.729/1.729/1.729/0.000 ms
```
Ok, looks good... But we can't forget the actual Internet Protocol, IPv6!!!
Let me show you our inet6.0 table:
```
root@R1> show route table inet6

inet6.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

fd10:faca:f0fa::1/128
                   *[Direct/0] 2w3d 23:03:54
                    >  via lo0.0
fd10:faca:f0fa::2/128
                   *[IS-IS/15] 00:16:42, metric 5
                    >  to fe80::2e6b:f5ff:fe89:bdc0 via ae0.0
fd10:faca:f0fa::3/128
                   *[IS-IS/15] 00:16:18, metric 15
                    >  to fe80::2e6b:f5ff:fe89:bdc0 via ae0.0
fd10:faca:f0fa::5/128
                   *[IS-IS/15] 00:15:53, metric 20
                    >  to fe80::528b:d8ff:fe00:1205 via ge-0/0/3.0
fd10:faca:f0fa::6/128
                   *[IS-IS/15] 00:15:38, metric 25
                    >  to fe80::2e6b:f5ff:fe89:bdc0 via ae0.0
                       to fe80::528b:d8ff:fe00:1205 via ge-0/0/3.0
fd10:faca:f0fa::7/128
                   *[IS-IS/15] 00:15:28, metric 15
                    >  to fe80::2e6b:f5ff:fe89:bdc0 via ae0.0
fd10:faca:f0fa::8/128
                   *[IS-IS/15] 00:15:11, metric 10
                    >  to fe80::528b:d8ff:fe00:1205 via ge-0/0/3.0
```
Are you missing something? No? Sure? 
Where is the IPv6 address of R4? Where is him? 

But this address are in the isis database, but why it is not in our routing table? 

I know the answer... When we are using a dual-stack IS-IS, the routers make only one SPF calculation, and using the IPv4 tree, but you remember our topology? In the transversal links we don't have family inet6 configured. 
In the R1 view, the best route for fd10:faca:f0fa::4/128 is via R4 link, but in this interface we don't have inet6 configured, then, the route becomes inactive. 

For resolve this, we need to make a second SPF calculation, and we can do this adding another topology in the IS-IS, then, the routers will make two SPF calculations, one for IPv4 topology and another one for IPv6 topology, let's do this in all routers of our topology:
```
set protocols isis topologies ipv6-unicast
```
Applying this in our topology, the connectivy will be ok, end-to-end. Let's check this:
```
root@R1> show route table inet6

inet6.0: 22 destinations, 22 routes (22 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

fd10:faca:f0fa::1/128
                   *[Direct/0] 2w3d 23:12:51
                    >  via lo0.0
fd10:faca:f0fa::2/128
                   *[IS-IS/15] 00:00:37, metric 5
                    >  to fe80::2e6b:f5ff:fe89:bdc0 via ae0.0
fd10:faca:f0fa::3/128
                   *[IS-IS/15] 00:00:37, metric 15
                    >  to fe80::2e6b:f5ff:fe89:bdc0 via ae0.0
fd10:faca:f0fa::4/128
                   *[IS-IS/15] 00:00:37, metric 25
                    >  to fe80::2e6b:f5ff:fe89:bdc0 via ae0.0
fd10:faca:f0fa::5/128
                   *[IS-IS/15] 00:00:37, metric 35
                    >  to fe80::2e6b:f5ff:fe89:bdc0 via ae0.0
                       to fe80::528b:d8ff:fe00:1205 via ge-0/0/3.0
fd10:faca:f0fa::6/128
                   *[IS-IS/15] 00:01:32, metric 30
                    >  to fe80::528b:d8ff:fe00:1205 via ge-0/0/3.0
fd10:faca:f0fa::7/128
                   *[IS-IS/15] 00:01:32, metric 20
                    >  to fe80::528b:d8ff:fe00:1205 via ge-0/0/3.0
fd10:faca:f0fa::8/128
                   *[IS-IS/15] 00:01:32, metric 10
                    >  to fe80::528b:d8ff:fe00:1205 via ge-0/0/3.0

root@R1> ping count 1 fd10:faca:f0fa::2
PING6(56=40+8+8 bytes) fd10:faca:f0fa::1 --> fd10:faca:f0fa::2
16 bytes from fd10:faca:f0fa::2, icmp_seq=0 hlim=64 time=2.858 ms

--- fd10:faca:f0fa::2 ping6 statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 2.858/2.858/2.858/0.000 ms

root@R1> ping count 1 fd10:faca:f0fa::3
PING6(56=40+8+8 bytes) fd10:faca:f0fa::1 --> fd10:faca:f0fa::3
16 bytes from fd10:faca:f0fa::3, icmp_seq=0 hlim=63 time=4.527 ms

--- fd10:faca:f0fa::3 ping6 statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 4.527/4.527/4.527/0.000 ms

root@R1> ping count 1 fd10:faca:f0fa::4
PING6(56=40+8+8 bytes) fd10:faca:f0fa::1 --> fd10:faca:f0fa::4
16 bytes from fd10:faca:f0fa::4, icmp_seq=0 hlim=62 time=3.920 ms

--- fd10:faca:f0fa::4 ping6 statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 3.920/3.920/3.920/0.000 ms

root@R1> ping count 1 fd10:faca:f0fa::5
PING6(56=40+8+8 bytes) fd10:faca:f0fa::1 --> fd10:faca:f0fa::5
16 bytes from fd10:faca:f0fa::5, icmp_seq=0 hlim=61 time=5.298 ms

--- fd10:faca:f0fa::5 ping6 statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 5.298/5.298/5.298/0.000 ms

root@R1> ping count 1 fd10:faca:f0fa::6
PING6(56=40+8+8 bytes) fd10:faca:f0fa::1 --> fd10:faca:f0fa::6
16 bytes from fd10:faca:f0fa::6, icmp_seq=0 hlim=62 time=4.317 ms

--- fd10:faca:f0fa::6 ping6 statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 4.317/4.317/4.317/0.000 ms

root@R1> ping count 1 fd10:faca:f0fa::7
PING6(56=40+8+8 bytes) fd10:faca:f0fa::1 --> fd10:faca:f0fa::7
16 bytes from fd10:faca:f0fa::7, icmp_seq=0 hlim=63 time=5.384 ms

--- fd10:faca:f0fa::7 ping6 statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 5.384/5.384/5.384/0.000 ms

root@R1> ping count 1 fd10:faca:f0fa::8
PING6(56=40+8+8 bytes) fd10:faca:f0fa::1 --> fd10:faca:f0fa::8
16 bytes from fd10:faca:f0fa::8, icmp_seq=0 hlim=64 time=4.802 ms

--- fd10:faca:f0fa::8 ping6 statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 4.802/4.802/4.802/0.000 ms
```
Now, everything is good!!! 

