# Flow Sampling and gRPC for Telemetry Transmission

Hello guys,

Today, we'll configure the flow sampling in our R5, to capture some packtes from DC3 (This is a new guy in our topology, but I won"t introduce you now). 

I'll use IPFIX to sent the flow samples to our SRV1, we'll configure a new interface between SRV1 and R5 for this communication. 
So, first, we need to have communication with our SRV1, I have separated the network 10.10.11.0/24 for this, I have some plans to distribute a DNS server to our customer in this network also. But in this first moment, I'll use this network to communicate the R5 with the SRV1 and send the flow samples.
```
set interfaces ge-0/0/4 description to-SRV1/eth1
set interfaces ge-0/0/4 unit 0 family inet address 10.10.11.1/24

root@R5> ping 10.10.11.2
PING 10.10.11.2 (10.10.11.2): 56 data bytes
64 bytes from 10.10.11.2: icmp_seq=0 ttl=64 time=159.553 ms
64 bytes from 10.10.11.2: icmp_seq=1 ttl=64 time=1.615 ms
64 bytes from 10.10.11.2: icmp_seq=2 ttl=64 time=2.168 ms
64 bytes from 10.10.11.2: icmp_seq=3 ttl=64 time=1.843 ms
^C
--- 10.10.11.2 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.615/41.295/159.553/68.277 ms
```
Ok, we have communication between them. 
Now, we have to made some templates for the families, inet and inet6. 
The configuration is as follows: 

Each flow is considered active, if it was 30 seconds. So, we need 30 seconds of pings for this traffic is considered a flow. 

Each flow is considered inactive if we no have this traffic for 180 seconds. So, for this traffic is considered active again after considered inactive, it need to be constant for 30 seconds.

We we'll make the same template configuration for the two families, let's go. 
```
set services flow-monitoring version-ipfix template SRV1-IPv6-IPFIX flow-active-timeout 30
set services flow-monitoring version-ipfix template SRV1-IPv6-IPFIX flow-inactive-timeout 180
set services flow-monitoring version-ipfix template SRV1-IPv6-IPFIX ipv6-template
set services flow-monitoring version-ipfix template SRV1-IPv4-IPFIX flow-active-timeout 30
set services flow-monitoring version-ipfix template SRV1-IPv4-IPFIX flow-inactive-timeout 180
set services flow-monitoring version-ipfix template SRV1-IPv4-IPFIX ipv4-template
```
Ok, we have the parameters of the template defined. Now, we need to made the configuration of the instance of sampling in the forwarding-options. 
We'll define the rate of the sampling in 1000, so, 1 packet in 1000 will be sampled, and we can improve the perfomance of sampling setting this function for the PFE, it is called inline-jflow. With this, the sampling will be sent directly by the PFE!!!

Another parameter that will be particular in our project as the destination port, the inet samplings will be send for the 44321 UDP, and the inet6 samplings to the 46321 UDP. 
With this in the mindset, we can go to the configuration. You can see that we need to specify the source address and the template here, but I think you understand this naturally considering you knowledge in Junos.
```
set forwarding-options sampling instance to-SRV1 input rate 1000
set forwarding-options sampling instance to-SRV1 family inet output flow-server 10.10.11.2 port 44321
set forwarding-options sampling instance to-SRV1 family inet output flow-server 10.10.11.2 source-address 10.0.0.5
set forwarding-options sampling instance to-SRV1 family inet output flow-server 10.10.11.2 version-ipfix template SRV1-IPv4-IPFIX
set forwarding-options sampling instance to-SRV1 family inet output inline-jflow source-address 10.0.0.5
set forwarding-options sampling instance to-SRV1 family inet6 output flow-server 10.10.11.2 port 46321
set forwarding-options sampling instance to-SRV1 family inet6 output flow-server 10.10.11.2 source-address 10.0.0.5
set forwarding-options sampling instance to-SRV1 family inet6 output flow-server 10.10.11.2 version-ipfix template SRV1-IPv6-IPFIX
set forwarding-options sampling instance to-SRV1 family inet6 output inline-jflow source-address 10.0.0.5
```
Ok, everything looks good now. But we need to associate this template for the FPC where the interface is present, and specify in which interface we want to collect the flows. 
```
set interfaces ge-0/0/6 description to-DC3/ge-0/0/1
set interfaces ge-0/0/6 unit 0 family inet sampling input
set interfaces ge-0/0/6 unit 0 family inet sampling output
set interfaces ge-0/0/6 unit 0 family inet address 172.30.130.5/30
set interfaces ge-0/0/6 unit 0 family inet6 sampling input
set interfaces ge-0/0/6 unit 0 family inet6 sampling output
set chassis fpc 0 sampling-instance to-SRV1
```
Ok, this way, we can see the flows arriving in the SRV1!
Let's make some pings to see. 
```
root@kvm:/usr/sbin# tcpdump -i eth1 'port 46321 or port 44321' -vvv -X
tcpdump: listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
09:39:21.558154 IP (tos 0x0, ttl 250, id 57810, offset 0, flags [none], proto UDP (17), length 138)
    10.0.0.5.50101 > 10.10.11.2.44321: [no cksum] UDP, length 110
        0x0000:  4500 008a e1d2 0000 fa11 bf7f 0a00 0005  E...............
        0x0010:  0a0a 0b02 c3b5 ad21 0076 0000 000a 006e  .......!.v.....n
        0x0020:  6939 69f8 0000 0009 0008 0000 0100 005e  i9i............^
        0x0030:  ac1e 8205 ac1e 8206 0001 0000 0000 0800  ................
        0x0040:  0000 0233 0000 201e 0000 fdfc 0000 fdfc  ...3............
        0x0050:  0000 0000 0000 0002 3340 4002 0400 0000  ........3@@.....
        0x0060:  00ff 0000 0000 0000 0000 0000 0000 0000  ................
        0x0070:  0054 0000 0000 0000 0001 0000 019b 0845  .T.............E
        0x0080:  7c00 0000 019b 0845 7c00                 |......E|.
09:39:25.640754 IP (tos 0x0, ttl 250, id 6000, offset 0, flags [none], proto UDP (17), length 138)
    10.0.0.5.50101 > 10.10.11.2.44321: [no cksum] UDP, length 110
        0x0000:  4500 008a 1770 0000 fa11 89e2 0a00 0005  E....p..........
        0x0010:  0a0a 0b02 c3b5 ad21 0076 0000 000a 006e  .......!.v.....n
        0x0020:  6939 69fc 0000 000a 0008 0000 0100 005e  i9i............^
        0x0030:  ac1e 8205 ac1e 8206 0001 0000 0000 0000  ................
        0x0040:  0000 0000 0000 201e 0000 fdfc 0000 fdfc  ................
        0x0050:  0000 0000 0000 0002 3340 4002 0400 0000  ........3@@.....
        0x0060:  00ff 0000 0000 0000 0000 0000 0000 0000  ................
        0x0070:  0054 0000 0000 0000 0001 0000 019b 0845  .T.............E
        0x0080:  8d00 0000 019b 0845 8d00                 .......E..
09:39:35.627258 IP (tos 0x0, ttl 250, id 13953, offset 0, flags [none], proto UDP (17), length 189)
    10.0.0.5.50102 > 10.10.11.2.46321: [no cksum] UDP, length 161
        0x0000:  4500 00bd 3681 0000 fa11 6a9e 0a00 0005  E...6.....j.....
        0x0010:  0a0a 0b02 c3b6 b4f1 00a9 0000 000a 00a1  ................
        0x0020:  6939 6a06 0000 00e3 0009 0000 0101 0091  i9j.............
        0x0030:  fe80 0000 0000 0000 52c8 d8ff fe00 0f08  ........R.......
        0x0040:  fe80 0000 0000 0000 0205 86ff fe71 8d01  .............q..
        0x0050:  003a 0000 0000 8000 0000 0233 0000 8000  .:.........3....
        0x0060:  0000 fdfc ffff ffff 0000 0000 0000 0000  ................
        0x0070:  0000 0000 0000 0000 0000 0000 0000 0000  ................
        0x0080:  0000 0000 0000 0000 0000 0000 0040 4002  .............@@.
        0x0090:  ff00 0000 0000 0000 0000 0000 0000 0000  ................
        0x00a0:  0000 0001 1800 0000 0000 0000 0500 0001  ................
        0x00b0:  9b08 45b3 0000 0001 9b08 45be 00         ..E.......E..
09:39:37.668597 IP (tos 0x0, ttl 250, id 65328, offset 0, flags [none], proto UDP (17), length 189)
    10.0.0.5.50102 > 10.10.11.2.46321: [no cksum] UDP, length 161
        0x0000:  4500 00bd ff30 0000 fa11 a1ee 0a00 0005  E....0..........
        0x0010:  0a0a 0b02 c3b6 b4f1 00a9 0000 000a 00a1  ................
        0x0020:  6939 6a08 0000 00e4 0009 0000 0101 0091  i9j.............
        0x0030:  fe80 0000 0000 0000 52c8 d8ff fe00 0f08  ........R.......
        0x0040:  fe80 0000 0000 0000 0205 86ff fe71 8d01  .............q..
        0x0050:  003a 0000 0000 8100 0000 0000 0000 8000  .:..............
        0x0060:  0000 fdfc ffff ffff 0000 0000 0000 0000  ................
        0x0070:  0000 0000 0000 0000 0000 0000 0000 0000  ................
        0x0080:  0000 0000 0000 0000 0000 0000 0040 4002  .............@@.
        0x0090:  ff00 0000 0000 0000 0000 0000 0000 0000  ................
        0x00a0:  0000 0000 7000 0000 0000 0000 0200 0001  ....p...........
        0x00b0:  9b08 45ba 0000 0001 9b08 45bc 00         ..E.......E..
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
```
And... Voilâ, we are receiving the samplings of inet and inet6 also. In this lab I don't want to waste time with applications of IPFIX, so only a tcpdump for me is good. 

Ok, now we are going for the telemetry... I want to learn more of telemetry for automation, and this is essential. 
