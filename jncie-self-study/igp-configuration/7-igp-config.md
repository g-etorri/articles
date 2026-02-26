# IGP Configuration

Hello everyone!

Today is the big day. It's time to bring up our IGP.

Our goals for today are:

* Establish connectivity with our new sites, DC2 and DC3.

* Achieve full IPv4 and IPv6 connectivity across our routers.

* Ensure full reachability between our Data Centers and the IGP core.

* That's the gist of it, with a few technical details thrown in.

* For our IGP backbone, we'll be running IS-IS.

* However, to keep things interesting (and for practice purposes), we'll mix it up:

* Connection to DC2 will be via BGP.

* Connection to DC3 will use OSPFv3.

To get full reachability, we'll need to redistribute (or leak) routes between these protocols and IS-IS, and vice-versa.

With that game plan in mind, here we go. This is our topology as it stands today:
<img width="1230" height="790" alt="image" src="https://github.com/user-attachments/assets/181ac158-9831-47d6-b873-e54394b24f3b" />

First things first, let's light up the new interfaces connecting to our DCs.

We'll stick to the addressing plan below:
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
You want to check connectivity, right? Just to put your mind at ease... I know, I know...
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

Alright, let's keep moving. Now, it's time to kick off the IS-IS configuration.

To get this running, we need to configure the Network Entity Title (NET) address, and enable family iso on our backbone interfaces.

To keep things organized, let's map out the NET addresses first:
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

If you recall the loopback addresses of our routers, the NET address logic will make perfect sense. It’s a standard best practice to use the Loopback IP as the Router ID and also as the base for the NET System ID.

For example, R1 has the loopback 10.0.0.1.

We can leverage the NET structure to embed this address right into the System ID.

The NET structure is: {AFI}.{AREA}.{SYSTEM-ID}.{NSEL}

So, breaking it down:
* AFI - 49
* AREA - 0001
* SYSTEM-ID - IP Address = 010.000.000.001 / SYS-ID = 0100.0000.0001
* NSEL - 0

Result = 49.0001.0100.0000.0001.00

Crystal clear, right? Now, let's configure the NET address and explicitly set the global router ID. I'll show R1's configuration only to keep things concise.
```
set interfaces lo0 unit 0 family iso address 49.0001.0100.0000.0001.00
set routing-options router-id 10.0.0.1
```
Now, for the backbone interfaces, let's use groups in Junos.

With this feature, we can apply the group, and the interface will inherit the configuration settings.

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

As I said, the interface inherits the configuration. In Junos, you can view this inheritance using a specific command:
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
So cool, right? Configuration groups are a lifesaver for keeping configs clean and saving lines!

Now, let's tackle the IS-IS configuration.

We’ll stick to a Level 1, single-area design. To support Traffic Engineering and larger metric values, we’ll enable wide-metrics.

To simulate best security practices, we'll apply MD5 authentication on adjacencies. For failure detection, we’re deploying BFD on all backbone interfaces with a 2s minimum interval and a multiplier of 3.

Quick Note: I know, I know... 2 seconds sounds like an eternity in a real backbone (we usually talk in milliseconds!). But since this is a virtual lab environment, we need to be gentle.

To optimize our IGP costs, we'll set the reference bandwidth to 10G. This ensures our aggregated interfaces (ae) get a better cost metric than single links.

Finally, here is a cool Junos trick: Since IS-IS strictly requires family iso to run, we can simply configure interface all under the protocol. Junos is smart enough to only enable IS-IS on the interfaces that actually have family iso configured. Huge time saver!
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

With this configuration, all interfaces with family iso are considered IS-IS interfaces:
```
root@R1> show isis interface
IS-IS interface database:
Interface             L CirID Level 1 DR        Level 2 DR        L1/L2 Metric
ae0.0                 1   0x1 Point to Point    Disabled                5/5
ge-0/0/2.0            1   0x1 Point to Point    Disabled               10/10
ge-0/0/3.0            1   0x1 Point to Point    Disabled               10/10
lo0.0                 1   0x1 Passive           Passive                 0/0
```
Take a look at the lo0.0 interface. It is passive by default, so we don't even need to bother setting it manually. Junos handles that for us.

Now, we need to push this configuration to all routers in our network.

Challenge time: I bet you can do this in under 5 minutes (if you are following along with the lab, of course!).

Ready? Let's check our IS-IS database now:
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
Everything looks good on paper, right?

But are you sure everything is okay? Are you really sure?

<img width="220" height="234" alt="image" src="https://github.com/user-attachments/assets/a9d1899c-7dff-4741-922d-77b467bc7987" />

Well, there is only one way to find out. Let's check the connectivity...
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
Ok, looks good... But we can't forget the modern Internet Protocol: IPv6!!!

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
Are you missing something? No? Are you sure? Where is the IPv6 address of R4? Where is it?

This address is in the IS-IS database, but why is it not in our routing table?

I know the answer... When we use default IS-IS (Single Topology), the routers make only one SPF calculation using the IPv4 tree. But remember our topology? In the transversal links, we don't have family inet6 configured.

In R1's view, the best route for fd10:faca:f0fa::4/128 is via the R4 link. However, since we don't have inet6 configured on this interface, the route becomes inactive.

To resolve this, we need to force a second SPF calculation by adding a secondary topology to IS-IS. Then, the routers will make two SPF calculations: one for the IPv4 topology and another for the IPv6 topology.

Let's do this on all routers:
```
set protocols isis topologies ipv6-unicast
```
Applying this to our topology, the connectivity should be ok, end-to-end. Let's check this:
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
Now, everything is good! But we haven't finished yet.

Let's complete the connection with our DCs.

DC1 is ok; we have connections with it and two gateways with VRRP configured. But our IGP cannot reach DC1.

Let's solve this by leaking the routes into the IGP on both routers, R3 and R4, to have redundancy:
R3:
```
set protocols isis interface ge-0/0/0.110 passive
set protocols isis interface ge-0/0/0.111 passive
```
R4:
```
set protocols isis interface ge-0/0/0.110 passive
set protocols isis interface ge-0/0/0.111 passive
```
Here, we don't need any policy to export these routes, considering they are direct connected. So, we only need to set these interfaces as passive in IS-IS.

Pro tip: Using passive interfaces is preferred over redistributing static/direct routes because IS-IS treats them as internal reachability (native) rather than external.

Ok, let's check the IS-IS database now:
```
root@R1> show isis database detail
IS-IS level 1 link-state database:
ISIS instance: Default, Routing Instance: master
...
R3.00-00 Sequence: 0x791, Checksum: 0xc47b, Lifetime: 1003 secs
   IPV4 Unicast IS neighbor: R2.00            Metric:       10
   IPV4 Unicast IS neighbor: R4.00            Metric:       10
   IPV4 Unicast IS neighbor: R6.00            Metric:       10
   IPV6 Unicast IS neighbor: R2.00            Metric:       10
   IPV6 Unicast IS neighbor: R4.00            Metric:       10
   IP IPV4 Unicast prefix: 10.0.0.3/32        Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.6/31      Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.10/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.12/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.110.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.111.0/24    Metric:       10 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::3/128 Metric:        0 Internal Up

R4.00-00 Sequence: 0x86e, Checksum: 0xcf22, Lifetime: 1194 secs
   IPV4 Unicast IS neighbor: R1.00            Metric:       10
   IPV4 Unicast IS neighbor: R3.00            Metric:       10
   IPV4 Unicast IS neighbor: R5.00            Metric:       10
   IPV6 Unicast IS neighbor: R3.00            Metric:       10
   IPV6 Unicast IS neighbor: R5.00            Metric:       10
   IP IPV4 Unicast prefix: 10.0.0.4/32        Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.2/31      Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.10/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.14/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.110.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.111.0/24    Metric:       10 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::4/128 Metric:        0 Internal Up
```
And... OK! We are receiving these routes in the IGP. Let's check the connectivity:
```
root@DC1> ping count 1 10.0.0.1
PING 10.0.0.1 (10.0.0.1): 56 data bytes
64 bytes from 10.0.0.1: icmp_seq=0 ttl=63 time=2.809 ms

--- 10.0.0.1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.809/2.809/2.809/0.000 ms

root@DC1> ping count 1 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: icmp_seq=0 ttl=63 time=1.859 ms

--- 10.0.0.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.859/1.859/1.859/0.000 ms

root@DC1> ping count 1 10.0.0.3
PING 10.0.0.3 (10.0.0.3): 56 data bytes
64 bytes from 10.0.0.3: icmp_seq=0 ttl=64 time=1.135 ms

--- 10.0.0.3 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.135/1.135/1.135/0.000 ms

root@DC1> ping count 1 10.0.0.4
PING 10.0.0.4 (10.0.0.4): 56 data bytes
64 bytes from 10.0.0.4: icmp_seq=0 ttl=64 time=1.200 ms

--- 10.0.0.4 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.200/1.200/1.200/0.000 ms

root@DC1> ping count 1 10.0.0.5
PING 10.0.0.5 (10.0.0.5): 56 data bytes
64 bytes from 10.0.0.5: icmp_seq=0 ttl=63 time=2.975 ms

--- 10.0.0.5 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.975/2.975/2.975/0.000 ms

root@DC1> ping count 1 10.0.0.6
PING 10.0.0.6 (10.0.0.6): 56 data bytes
64 bytes from 10.0.0.6: icmp_seq=0 ttl=63 time=1.348 ms

--- 10.0.0.6 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.348/1.348/1.348/0.000 ms

root@DC1> ping count 1 10.0.0.7
PING 10.0.0.7 (10.0.0.7): 56 data bytes
64 bytes from 10.0.0.7: icmp_seq=0 ttl=62 time=2.031 ms

--- 10.0.0.7 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.031/2.031/2.031/0.000 ms

root@DC1> ping count 1 10.0.0.8
PING 10.0.0.8 (10.0.0.8): 56 data bytes
64 bytes from 10.0.0.8: icmp_seq=0 ttl=62 time=7.884 ms

--- 10.0.0.8 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 7.884/7.884/7.884/0.000 ms
```
Success!

Now for DC2. We'll spin up two BGP sessions: one on R4 and another on R5. The plan is to export only a default route to DC2.

Here is a JNCIE pro-tip: Generally, in the exam, you aren't allowed to use static routes. So, to generate a default route cleanly, I'll use an aggregate route instead.

Oh, I almost forgot to tell you our Autonomous System Number (ASN): we are running AS 65020.

Let's get this configured:
R4:
```
set routing-options autonomous-system 65020
set routing-options aggregate route 0.0.0.0/0 discard
set protocols bgp group eBGP-AS64666-DC2 type external
set protocols bgp group eBGP-AS64666-DC2 description eBGP-AS64666-DC2
set protocols bgp group eBGP-AS64666-DC2 export Saida_DC2
set protocols bgp group eBGP-AS64666-DC2 peer-as 64666
set protocols bgp group eBGP-AS64666-DC2 neighbor 172.30.120.2 family inet unicast
set policy-options policy-statement Export_DC2 term Default from protocol aggregate
set policy-options policy-statement Export_DC2 term Default from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Export_DC2 term Default then accept
set policy-options policy-statement Export_DC2 then reject
```
But here we have a little problem: the default route is not active in DC2...
```
root@DC2> show route

inet.0: 25 destinations, 25 routes (25 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.102/32      *[Direct/0] 1w2d 00:50:51
                    > via lo0.0

But I'm exporting the default route for him:
root@R4> show route advertising-protocol bgp 172.30.120.2

inet.0: 58 destinations, 61 routes (58 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 0.0.0.0/0               Self                                    {64666} I
```
Ok, did you spot the problem? Did you check the AS-PATH?

Yesss!!! That is exactly the issue with using aggregate routes here.

If a BGP route contributes to the aggregate route, the AS-PATH attributes are inherited by default. Consequently, the BGP protocol on the receiving end drops this route because it identifies a routing loop (it sees its own AS in the path).

To resolve this, we can explicitly define the aggregator attributes for this route. Let me show you:
```
set routing-options aggregate route 0.0.0.0/0 as-path aggregator 65020 10.0.0.4
```
This way, the route is generated with AS 65020 and the Aggregator ID set to R4's Router ID.

We must replicate this on R5 to ensure redundancy.
R5:
```
set routing-options autonomous-system 65020
set routing-options aggregate route 0.0.0.0/0 as-path aggregator 65020 10.0.0.5
set routing-options aggregate route 0.0.0.0/0 discard
set protocols bgp group eBGP-AS64666-DC2 type external
set protocols bgp group eBGP-AS64666-DC2 description eBGP-AS64666-DC2
set protocols bgp group eBGP-AS64666-DC2 export Export_DC2
set protocols bgp group eBGP-AS64666-DC2 peer-as 64666
set protocols bgp group eBGP-AS64666-DC2 neighbor 172.30.120.6 family inet unicast
set policy-options policy-statement Export_DC2 term Default from protocol aggregate
set policy-options policy-statement Export_DC2 term Default from route-filter 0.0.0.0/0 exact
set policy-options policy-statement Export_DC2 term Default then accept
set policy-options policy-statement Export_DC2 then reject
```

Ok, halfway there (1/2)!

We need to ensure our IGP can actually reach DC2 right now.

To do this, we must redistribute the BGP routes learned from DC2 into our IS-IS backbone. Unlike the direct interfaces we saw earlier, for BGP we definitely need a routing policy.

Here is the configuration for R4 and R5:
```
set policy-options policy-statement redistribute-isis term BGP from protocol bgp
set policy-options policy-statement redistribute-isis term BGP then accept
set protocols isis export redistribute-isis
```
Now, the moment of truth!

We need to check our IS-IS database to confirm that we are receiving the BGP routes from DC2:
```
R4.00-00 Sequence: 0x86f, Checksum: 0xc939, Lifetime: 1183 secs
   IPV4 Unicast IS neighbor: R1.00            Metric:       10
   IPV4 Unicast IS neighbor: R3.00            Metric:       10
   IPV4 Unicast IS neighbor: R5.00            Metric:       10
   IPV6 Unicast IS neighbor: R3.00            Metric:       10
   IPV6 Unicast IS neighbor: R5.00            Metric:       10
   IP IPV4 Unicast prefix: 10.0.0.4/32        Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.2/31      Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.10/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.14/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.110.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.111.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.120.4/30    Metric:       10 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::4/128 Metric:        0 Internal Up

R5.00-00 Sequence: 0x868, Checksum: 0x8f1d, Lifetime: 1189 secs
   IPV4 Unicast IS neighbor: R4.00            Metric:       10
   IPV4 Unicast IS neighbor: R6.00            Metric:        5
   IPV4 Unicast IS neighbor: R8.00            Metric:       10
   IPV6 Unicast IS neighbor: R4.00            Metric:       10
   IPV6 Unicast IS neighbor: R6.00            Metric:        5
   IP IPV4 Unicast prefix: 10.0.0.5/32        Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 10.0.0.102/32      Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.14/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.16/31     Metric:        5 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.18/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.120.0/30    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.220.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.221.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.222.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.223.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.224.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.225.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.226.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.227.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.228.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.229.0/24    Metric:       10 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::5/128 Metric:        0 Internal Up
```
Only R5 is exporting the BGP routes into IS-IS. But why?

I'll explain, after all, I'm the one writing this. 

It’s a classic race condition: R5 was the first router to export these routes into the IGP. Consequently, R4 received the update and installed the IS-IS route in its routing table instead of the route received via BGP.

That's because, in Junos, IS-IS routes have a better preference than BGP:
```
root@R4> show route receive-protocol bgp 172.30.120.2

inet.0: 58 destinations, 71 routes (58 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  10.0.0.102/32           172.30.120.2                            64666 I
  172.30.120.0/30         172.30.120.2                            64666 I
* 172.30.120.4/30         172.30.120.2                            64666 I
  172.30.220.0/24         172.30.120.2                            64666 I
  172.30.221.0/24         172.30.120.2                            64666 I
  172.30.222.0/24         172.30.120.2                            64666 I
  172.30.223.0/24         172.30.120.2                            64666 I
  172.30.224.0/24         172.30.120.2                            64666 I
  172.30.225.0/24         172.30.120.2                            64666 I
  172.30.226.0/24         172.30.120.2                            64666 I
  172.30.227.0/24         172.30.120.2                            64666 I
  172.30.228.0/24         172.30.120.2                            64666 I
  172.30.229.0/24         172.30.120.2                            64666 I

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)

inet6.0: 24 destinations, 24 routes (24 active, 0 holddown, 0 hidden)

root@R4> show route 10.0.0.102/32

inet.0: 58 destinations, 71 routes (58 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.102/32      *[IS-IS/15] 00:14:18, metric 20
                    >  to 10.200.0.15 via ge-0/0/3.0
                    [BGP/170] 00:05:24, localpref 100
                      AS path: 64666 I, validation-state: unverified
                    >  to 172.30.120.2 via ge-0/0/1.0
```
To solve this, we can change the preference of the BGP protocol, but only for the DC2 group.

Let's set a preference value of 14 in both routers to prevent this problem and ensure redundancy and traffic load-balancing.

R4 and R5:
```
set protocols bgp group eBGP-AS64666-DC2 preference 14
```
Now, let's double-check everything.

We need to verify if the BGP routes are now winning the preference battle, getting installed in the routing table, and correctly exported to our IGP:
```
root@R4> show bgp summary
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                      13         12          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.30.120.2          64666         40         69       0       0       30:11 Establ
  inet.0: 12/13/13/0


root@R5> show bgp summary
Threading mode: BGP I/O
Default eBGP mode: advertise - accept, receive - accept
Groups: 1 Peers: 1 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                      13         12          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
172.30.120.6          64666         29         49       0       0       20:39 Establ
  inet.0: 12/13/13/0

R4.00-00 Sequence: 0x870, Checksum: 0xc47f, Lifetime: 1103 secs
   IPV4 Unicast IS neighbor: R1.00            Metric:       10
   IPV4 Unicast IS neighbor: R3.00            Metric:       10
   IPV4 Unicast IS neighbor: R5.00            Metric:       10
   IPV6 Unicast IS neighbor: R3.00            Metric:       10
   IPV6 Unicast IS neighbor: R5.00            Metric:       10
   IP IPV4 Unicast prefix: 10.0.0.4/32        Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 10.0.0.102/32      Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.2/31      Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.10/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.14/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.110.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.111.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.120.4/30    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.220.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.221.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.222.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.223.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.224.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.225.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.226.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.227.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.228.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.229.0/24    Metric:       10 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::4/128 Metric:        0 Internal Up

R5.00-00 Sequence: 0x868, Checksum: 0x8f1d, Lifetime: 854 secs
   IPV4 Unicast IS neighbor: R4.00            Metric:       10
   IPV4 Unicast IS neighbor: R6.00            Metric:        5
   IPV4 Unicast IS neighbor: R8.00            Metric:       10
   IPV6 Unicast IS neighbor: R4.00            Metric:       10
   IPV6 Unicast IS neighbor: R6.00            Metric:        5
   IP IPV4 Unicast prefix: 10.0.0.5/32        Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 10.0.0.102/32      Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.14/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.16/31     Metric:        5 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.18/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.120.0/30    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.220.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.221.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.222.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.223.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.224.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.225.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.226.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.227.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.228.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.229.0/24    Metric:       10 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::5/128 Metric:        0 Internal Up
```
And... Success!!! Now, let's confirm we have connectivity with everyone:
```
root@DC2> ping count 1 10.0.0.1
PING 10.0.0.1 (10.0.0.1): 56 data bytes
64 bytes from 10.0.0.1: icmp_seq=0 ttl=62 time=1.551 ms

--- 10.0.0.1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.551/1.551/1.551/0.000 ms

root@DC2> ping count 1 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: icmp_seq=0 ttl=61 time=1.984 ms

--- 10.0.0.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.984/1.984/1.984/0.000 ms

root@DC2> ping count 1 10.0.0.3
PING 10.0.0.3 (10.0.0.3): 56 data bytes
64 bytes from 10.0.0.3: icmp_seq=0 ttl=62 time=1.703 ms

--- 10.0.0.3 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.703/1.703/1.703/0.000 ms

root@DC2> ping count 1 10.0.0.4
PING 10.0.0.4 (10.0.0.4): 56 data bytes
64 bytes from 10.0.0.4: icmp_seq=0 ttl=64 time=0.846 ms

--- 10.0.0.4 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.846/0.846/0.846/0.000 ms

root@DC2> ping count 1 10.0.0.5
PING 10.0.0.5 (10.0.0.5): 56 data bytes
64 bytes from 10.0.0.5: icmp_seq=0 ttl=64 time=1.572 ms

--- 10.0.0.5 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.572/1.572/1.572/0.000 ms

root@DC2> ping count 1 10.0.0.6
PING 10.0.0.6 (10.0.0.6): 56 data bytes
64 bytes from 10.0.0.6: icmp_seq=0 ttl=63 time=1.796 ms

--- 10.0.0.6 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.796/1.796/1.796/0.000 ms

root@DC2> ping count 1 10.0.0.7
PING 10.0.0.7 (10.0.0.7): 56 data bytes
64 bytes from 10.0.0.7: icmp_seq=0 ttl=62 time=3.076 ms

--- 10.0.0.7 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 3.076/3.076/3.076/0.000 ms

root@DC2> ping count 1 10.0.0.8
PING 10.0.0.8 (10.0.0.8): 56 data bytes
64 bytes from 10.0.0.8: icmp_seq=0 ttl=63 time=2.024 ms

--- 10.0.0.8 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.024/2.024/2.024/0.000 ms

root@DC2> ping count 1 172.30.110.253
PING 172.30.110.253 (172.30.110.253): 56 data bytes
64 bytes from 172.30.110.253: icmp_seq=0 ttl=61 time=2.519 ms

--- 172.30.110.253 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.519/2.519/2.519/0.000 ms

root@DC2> ping count 1 172.30.111.253
PING 172.30.111.253 (172.30.111.253): 56 data bytes
64 bytes from 172.30.111.253: icmp_seq=0 ttl=61 time=2.774 ms

--- 172.30.111.253 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.774/2.774/2.774/0.000 ms
```
Everything is ok!!! We have connectivity with all routers of our IGP, and with DC1 too!

Now, let's go to DC3.

This DC will have IPv6 support also. So, we'll work with OSPFv3 to provide this. OSPFv3 speaks IPv6 natively; to make it speak IPv4, we need to configure the realm ipv4-unicast.

So, let's configure this on R4 and R5:
```
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface ge-0/0/5.0
set protocols ospf3 area 0.0.0.0 interface ge-0/0/5.0
```
Simple, right? Everything is working, right? So, so.

We need to make everything reachable in our network; therefore, we need to leak the OSPF routes into IS-IS and vice-versa.

Now, we'll make the configuration for the leak into IS-IS, but you can imagine the problem. Before, we saw the problem with the BGP routes leaked: the BGP preference is higher than IS-IS, so only one router was leaking the routes.

Here, we need to do the same thing. I'll make the configuration without changing the preference to prove it. I'll add a new term to the redistribute policy:
```
set policy-options policy-statement redistribute-isis term OSPF3 from protocol ospf3
set policy-options policy-statement redistribute-isis term OSPF3 then accept
```

Let's check our IS-IS database now:
```
R4.00-00 Sequence: 0xa4c, Checksum: 0x18a3, Lifetime: 1149 secs
   IPV4 Unicast IS neighbor: R1.00            Metric:       10
   IPV4 Unicast IS neighbor: R3.00            Metric:       10
   IPV4 Unicast IS neighbor: R5.00            Metric:       10
   IPV6 Unicast IS neighbor: R3.00            Metric:       10
   IPV6 Unicast IS neighbor: R5.00            Metric:       10
   IP IPV4 Unicast prefix: 10.0.0.4/32        Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 10.0.0.102/32      Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.0.0.103/32      Metric:        1 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.2/31      Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.10/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.14/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.110.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.111.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.120.4/30    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.130.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.130.4/30    Metric:        2 Internal Up
   IP IPV4 Unicast prefix: 172.30.131.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.132.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.133.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.134.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.135.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.136.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.137.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.138.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.139.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.220.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.221.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.222.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.223.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.224.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.225.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.226.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.227.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.228.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.229.0/24    Metric:       10 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::4/128 Metric:        0 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::103/128 Metric:        1 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:1::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:2::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:3::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:4::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:5::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:6::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:7::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:8::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:9::/80 Metric:        0 External Up

R5.00-00 Sequence: 0xa42, Checksum: 0x5028, Lifetime: 1179 secs
   IPV4 Unicast IS neighbor: R4.00            Metric:       10
   IPV4 Unicast IS neighbor: R6.00            Metric:        5
   IPV4 Unicast IS neighbor: R8.00            Metric:       10
   IPV6 Unicast IS neighbor: R4.00            Metric:       10
   IPV6 Unicast IS neighbor: R6.00            Metric:        5
   IP IPV4 Unicast prefix: 10.0.0.5/32        Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 10.0.0.102/32      Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.0.0.103/32      Metric:        1 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.14/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.16/31     Metric:        5 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.18/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.120.0/30    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.130.0/30    Metric:        2 Internal Up
   IP IPV4 Unicast prefix: 172.30.220.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.221.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.222.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.223.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.224.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.225.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.226.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.227.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.228.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.229.0/24    Metric:       10 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::5/128 Metric:        0 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::103/128 Metric:        1 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:1::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:2::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:3::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:4::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:5::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:6::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:7::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:8::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:9::/80 Metric:        0 External Up
```
You can see that the IPv4 routes aren't exported by R5. But you might ask: What about the IPv6 routes? Why are both routers exporting these routes?

This happens because the wide-metrics-only setting changes how the external IPv4 routes are processed, potentially making them look like internal routes (lower preference). IPv6, however, uses TLV 236 and is correctly tagged with the External Bit.

So, let's change the OSPF external preference to fix the IPv4 leak and ensure redundancy:
```
set protocols ospf3 realm ipv4-unicast external-preference 13
set protocols ospf3 external-preference 13
```
I am changing the preference for both families to create a standard. I like these 'hacks' to be a little bit beautiful.

And, let's check the results!
```
R4.00-00 Sequence: 0xa4c, Checksum: 0x18a3, Lifetime: 419 secs
   IPV4 Unicast IS neighbor: R1.00            Metric:       10
   IPV4 Unicast IS neighbor: R3.00            Metric:       10
   IPV4 Unicast IS neighbor: R5.00            Metric:       10
   IPV6 Unicast IS neighbor: R3.00            Metric:       10
   IPV6 Unicast IS neighbor: R5.00            Metric:       10
   IP IPV4 Unicast prefix: 10.0.0.4/32        Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 10.0.0.102/32      Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.0.0.103/32      Metric:        1 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.2/31      Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.10/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.14/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.110.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.111.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.120.4/30    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.130.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.130.4/30    Metric:        2 Internal Up
   IP IPV4 Unicast prefix: 172.30.131.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.132.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.133.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.134.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.135.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.136.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.137.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.138.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.139.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.220.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.221.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.222.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.223.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.224.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.225.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.226.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.227.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.228.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.229.0/24    Metric:       10 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::4/128 Metric:        0 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::103/128 Metric:        1 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:1::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:2::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:3::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:4::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:5::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:6::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:7::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:8::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:9::/80 Metric:        0 External Up

R5.00-00 Sequence: 0xa43, Checksum: 0xea7b, Lifetime: 1188 secs
   IPV4 Unicast IS neighbor: R4.00            Metric:       10
   IPV4 Unicast IS neighbor: R6.00            Metric:        5
   IPV4 Unicast IS neighbor: R8.00            Metric:       10
   IPV6 Unicast IS neighbor: R4.00            Metric:       10
   IPV6 Unicast IS neighbor: R6.00            Metric:        5
   IP IPV4 Unicast prefix: 10.0.0.5/32        Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 10.0.0.102/32      Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.0.0.103/32      Metric:        1 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.14/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.16/31     Metric:        5 Internal Up
   IP IPV4 Unicast prefix: 10.200.0.18/31     Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.120.0/30    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.130.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.130.0/30    Metric:        2 Internal Up
   IP IPV4 Unicast prefix: 172.30.131.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.132.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.133.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.134.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.135.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.136.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.137.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.138.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.139.0/24    Metric:        0 Internal Up
   IP IPV4 Unicast prefix: 172.30.220.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.221.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.222.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.223.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.224.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.225.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.226.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.227.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.228.0/24    Metric:       10 Internal Up
   IP IPV4 Unicast prefix: 172.30.229.0/24    Metric:       10 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::5/128 Metric:        0 Internal Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa::103/128 Metric:        1 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:1::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:2::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:3::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:4::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:5::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:6::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:7::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:8::/80 Metric:        0 External Up
   V6 IPV6 Unicast prefix: fd10:faca:f0fa:3:9::/80 Metric:        0 External Up
```
Fantastic! Both routers are now exporting the DC3 prefixes correctly into the IS-IS backbone.

However, we still lack full connectivity because the DC3 domain is blind to the rest of the core network.

The crucial next step is to enable two-way communication: we need to leak the IS-IS routes into the OSPFv3 domain.

We will create a policy to do this:
```
set policy-options policy-statement redistribute-ospf3 term ISIS from protocol isis
set policy-options policy-statement redistribute-ospf3 term ISIS then accept
```
But, as you know, OSPF is a dynamic protocol that floods LSAs to all routers in the network.

And now, thinking more about this, I can deduce that you have imagined the routing loop that we are having here...
```
root@R4> show route 10.0.0.0/24 active-path

inet.0: 58 destinations, 73 routes (58 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.1/32        *[OSPF3/13] 00:06:33, metric 20, tag 0
                    >  to 172.30.130.2 via ge-0/0/5.0
10.0.0.2/32        *[OSPF3/13] 00:06:33, metric 25, tag 0
                    >  to 172.30.130.2 via ge-0/0/5.0
10.0.0.3/32        *[OSPF3/13] 00:06:33, metric 15, tag 0
                    >  to 172.30.130.2 via ge-0/0/5.0
10.0.0.4/32        *[Direct/0] 3w0d 07:15:17
                    >  via lo0.0
10.0.0.5/32        *[IS-IS/15] 4d 06:50:18, metric 10
                    >  to 10.200.0.15 via ge-0/0/3.0
10.0.0.6/32        *[OSPF3/13] 00:06:33, metric 5, tag 0
                    >  to 172.30.130.2 via ge-0/0/5.0
10.0.0.7/32        *[OSPF3/13] 00:06:33, metric 15, tag 0
                    >  to 172.30.130.2 via ge-0/0/5.0
10.0.0.8/32        *[OSPF3/13] 00:06:33, metric 10, tag 0
                    >  to 172.30.130.2 via ge-0/0/5.0
```
If the R5 is exporting IS-IS routes into OSPFv3, and the OSPFv3 has a lower preference than IS-IS, the R4 will prefer the OSPFv3 routes, right? And the R4, preferring the OSPFv3 routes in its routing table, will export these routes back into IS-IS because it has a policy that makes this happen, and R5 will install some routes from R4, and the loop continues...

Now, to avoid this, we can use route tags!

We can tag the exported routes and reject those routes to avoid this loop.

Let's do it!
```
set policy-options policy-statement redistribute-ospf3 term ISIS then tag 123
set policy-options policy-statement import-ospf3 term drop-tag from protocol ospf3
set policy-options policy-statement import-ospf3 term drop-tag from tag 123
set policy-options policy-statement import-ospf3 term drop-tag then reject
set protocols ospf3 realm ipv4-unicast import import-ospf3
set protocols ospf3 import import-ospf3
```
This way, we are tagging the IS-IS routes exported into OSPFv3, and rejecting them on the other router. So, the loop is avoided. 
```
root@R4> show route 10.0.0.0/24 active-path

inet.0: 58 destinations, 61 routes (58 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.1/32        *[IS-IS/15] 4d 07:01:08, metric 10
                    >  to 10.200.0.2 via ge-0/0/2.0
10.0.0.2/32        *[IS-IS/15] 4d 07:01:08, metric 15
                    >  to 10.200.0.2 via ge-0/0/2.0
10.0.0.3/32        *[IS-IS/15] 4d 07:01:08, metric 10
                    >  to 10.200.0.10 via ge-0/0/4.0
10.0.0.4/32        *[Direct/0] 3w0d 07:26:07
                    >  via lo0.0
10.0.0.5/32        *[IS-IS/15] 4d 07:01:08, metric 10
                    >  to 10.200.0.15 via ge-0/0/3.0
10.0.0.6/32        *[IS-IS/15] 00:00:01, metric 15
                    >  to 10.200.0.15 via ge-0/0/3.0
10.0.0.7/32        *[IS-IS/15] 00:00:01, metric 25
                       to 10.200.0.2 via ge-0/0/2.0
                    >  to 10.200.0.15 via ge-0/0/3.0
10.0.0.8/32        *[IS-IS/15] 00:00:01, metric 20
                    >  to 10.200.0.2 via ge-0/0/2.0
                       to 10.200.0.15 via ge-0/0/3.0

root@R5> show route 10.0.0.0/24 active-path

inet.0: 58 destinations, 61 routes (58 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.1/32        *[IS-IS/15] 4d 07:01:29, metric 20
                       to 10.200.0.19 via ge-0/0/2.0
                    >  to 10.200.0.14 via ge-0/0/3.0
10.0.0.2/32        *[IS-IS/15] 4d 07:01:29, metric 25
                       to 10.200.0.19 via ge-0/0/2.0
                    >  to 10.200.0.14 via ge-0/0/3.0
                       to 10.200.0.17 via ae0.0
10.0.0.3/32        *[IS-IS/15] 4d 07:01:29, metric 15
                    >  to 10.200.0.17 via ae0.0
10.0.0.4/32        *[IS-IS/15] 4d 07:01:29, metric 10
                    >  to 10.200.0.14 via ge-0/0/3.0
10.0.0.5/32        *[Direct/0] 3w0d 06:45:04
                    >  via lo0.0
10.0.0.6/32        *[IS-IS/15] 4d 07:01:29, metric 5
                    >  to 10.200.0.17 via ae0.0
10.0.0.7/32        *[IS-IS/15] 4d 07:01:29, metric 15
                    >  to 10.200.0.17 via ae0.0
10.0.0.8/32        *[IS-IS/15] 4d 07:01:29, metric 10
                    >  to 10.200.0.19 via ge-0/0/2.0
```
Ok, now technically we have connectivity between our DC3 and our backbone. But, we want to make the connectivity end-to-end, so, we need to export the BGP routes of DC2 into the OSPFv3 domain.

This is easy, and we'll do the same thing that we did with the IS-IS routes. After this, we can test the total connectivity!

Let's go:
```
set policy-options policy-statement redistribute-ospf3 term BGP from protocol bgp
set policy-options policy-statement redistribute-ospf3 term BGP then tag 123
set policy-options policy-statement redistribute-ospf3 term BGP then accept
```

Ok, now we can perform a ping test for the router addresses:
```
root@DC3> ping count 1 10.0.0.1
PING 10.0.0.1 (10.0.0.1): 56 data bytes
64 bytes from 10.0.0.1: icmp_seq=0 ttl=63 time=4.011 ms

--- 10.0.0.1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 4.011/4.011/4.011/0.000 ms

root@DC3> ping count 1 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: icmp_seq=0 ttl=61 time=2.373 ms

--- 10.0.0.2 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.373/2.373/2.373/0.000 ms

root@DC3> ping count 1 10.0.0.3
PING 10.0.0.3 (10.0.0.3): 56 data bytes
64 bytes from 10.0.0.3: icmp_seq=0 ttl=62 time=1.416 ms

--- 10.0.0.3 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.416/1.416/1.416/0.000 ms

root@DC3> ping count 1 10.0.0.4
PING 10.0.0.4 (10.0.0.4): 56 data bytes
64 bytes from 10.0.0.4: icmp_seq=0 ttl=64 time=6.920 ms

--- 10.0.0.4 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 6.920/6.920/6.920/0.000 ms

root@DC3> ping count 1 10.0.0.5
PING 10.0.0.5 (10.0.0.5): 56 data bytes
64 bytes from 10.0.0.5: icmp_seq=0 ttl=64 time=1.619 ms

--- 10.0.0.5 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.619/1.619/1.619/0.000 ms

root@DC3> ping count 1 10.0.0.6
PING 10.0.0.6 (10.0.0.6): 56 data bytes
64 bytes from 10.0.0.6: icmp_seq=0 ttl=63 time=1.864 ms

--- 10.0.0.6 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.864/1.864/1.864/0.000 ms

root@DC3> ping count 1 10.0.0.7
PING 10.0.0.7 (10.0.0.7): 56 data bytes
64 bytes from 10.0.0.7: icmp_seq=0 ttl=61 time=2.298 ms

--- 10.0.0.7 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.298/2.298/2.298/0.000 ms

root@DC3> ping count 1 10.0.0.8
PING 10.0.0.8 (10.0.0.8): 56 data bytes
64 bytes from 10.0.0.8: icmp_seq=0 ttl=63 time=1.717 ms

--- 10.0.0.8 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.717/1.717/1.717/0.000 ms

root@DC3> ping count 1 10.0.0.102
PING 10.0.0.102 (10.0.0.102): 56 data bytes
64 bytes from 10.0.0.102: icmp_seq=0 ttl=63 time=80.355 ms

--- 10.0.0.102 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 80.355/80.355/80.355/0.000 ms

root@DC3> ping count 1 10.0.0.103
PING 10.0.0.103 (10.0.0.103): 56 data bytes
64 bytes from 10.0.0.103: icmp_seq=0 ttl=64 time=0.033 ms

--- 10.0.0.103 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.033/0.033/0.033/0.000 ms

root@DC3> ping count 1 172.30.110.253
PING 172.30.110.253 (172.30.110.253): 56 data bytes
64 bytes from 172.30.110.253: icmp_seq=0 ttl=62 time=2.150 ms

--- 172.30.110.253 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.150/2.150/2.150/0.000 ms

root@DC3> ping count 1 172.30.111.253
PING 172.30.111.253 (172.30.111.253): 56 data bytes
64 bytes from 172.30.111.253: icmp_seq=0 ttl=62 time=2.256 ms

--- 172.30.111.253 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 2.256/2.256/2.256/0.000 ms

root@DC3>
```
And with that, we have achieved full connectivity across our entire backbone!

We now have a solid, resilient topology ready to deliver transit and other valuable services to our customers, and, of course, connect to our upstream providers.

In the next post, we will configure our comprehensive BGP topology. See you soon!
