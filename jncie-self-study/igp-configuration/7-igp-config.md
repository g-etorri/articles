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

