# Interfaces Configuration

Hey everyone,

Today we'll configure the addresses of our backbone interfaces, along with the interface that connects to our DC1!
To avoid wasting time, let's get straight to it.
We'll use the topology below:
<img width="1211" height="747" alt="image" src="https://github.com/user-attachments/assets/51ef5469-08d4-4bed-a2d8-81fb1244a531" />

And we'll follow the parameters below.
Yeah, FACA FOFA address — or cute knife for the English speakers out there.
Thanks, IPv6… always giving us these beautiful hexadecimal possibilites.
| Router | Interface | IPv4 address | IPv6 address |
| ------ | --------- | ------------ | ------------ |
| R1 | ae0.0 | 10.200.0.0/31 | link-local |
| R1 | ge-0/0/2.0 | 10.200.0.2/31 | |
| R1 | ge-0/0/3.0 | 10.200.0.4/31 | |
| R1 | lo0.0 | 10.0.0.1/32 | fd10:faca:f0fa::1/128 |
| R2 | ae0.0 | 10.200.0.1/31 | link-local | 
| R2 | ge-0/0/3.0 | 10.200.0.6/31 | link-local |
| R2 | ge-0/0/2.0 | 10.200.0.8/31 | | 
| R2 | lo0.0 | 10.0.0.2/32 | fd10:faca:f0fa::2/128 |
| R3 | ge-0/0/3.0 | 10.200.0.7/31 | link-local |
| R3 | ge-0/0/4.0 | 10.200.0.10/31 | link-local |
| R3 | ge-0/0/2.0 | 10.200.0.12/31 | |
| R3 | lo0.0 | 10.0.0.3/32 | fd10:faca:f0fa::3/128 |
| R4 | ge-0/0/4.0 | 10.200.0.11/31 | link-local |
| R4 | ge-0/0/3.0 | 10.200.0.14/31 | link-local | 
| R4 | ge-0/0/2.0 | 10.200.0.3/31 | |
| R4 | lo0.0 | 10.0.0.4/32 | fd10:faca:f0fa::4/128 |
| R5 | ge-0/0/3.0 | 10.200.0.15/31 | link-local |
| R5 | ae0.0 | 10.200.0.16/31 | link-local | 
| R5 | ge-0/0/2.0 | 10.200.0.18/31 | |
| R5 | lo0.0 | 10.0.0.5/32 | fd10:faca:f0fa::5/128 |
| R6 | ae0.0 | 10.200.0.17/31 | link-local |
| R6 | ge-0/0/2.0 | 10.200.0.13/31 | |
| R6 | ge-0/0/3.0 | 10.200.0.20/31 | link-local |
| R6 | lo0.0 | 10.0.0.6/32 | fd10:faca:f0fa::6/128 |
| R7 | ge-0/0/3.0 | 10.200.0.21/31 | link-local |
| R7 | ge-0/0/2.0 | 10.200.0.9/31 | |
| R7 | ge-0/0/4.0 | 10.200.0.22/31 | link-local |
| R7 | lo0.0 | 10.0.0.7/32 | fd10:faca:f0fa::7/128 |
| R8 | ge-0/0/4.0 | 10.200.0.23/31 | link-local |
| R8 | ge-0/0/3.0 | 10.200.0.5/31 | link-local |
| R8 | ge-0/0/2.0 | 10.200.0.19/31 | |
| R8 | lo0.0 | 10.0.0.8/32 | fd10:faca:f0fa::8/128 |

Ok, the first thing we need to do here is create the LACP interfaces, a.k.a. "ae" or aggregated Ethernet interfaces.
So we’ll configure this on the links between R1–R2 and R5–R6. Let’s get to it.

First, we need to enable LACP on the chassis so it can actually work:
```
set chassis aggregated-devices ethernet device-count 4
```
With this enabled, we can now configure the physical interfaces and the ae interfaces:
```
set interfaces ge-0/0/0 description to-R2/ge-0/0/0
set interfaces ge-0/0/0 gigether-options 802.3ad ae0
set interfaces ge-0/0/1 description to-R2/ge-0/0/1
set interfaces ge-0/0/1 gigether-options 802.3ad ae0
set interfaces ae0 description to-R2/ae0
set interfaces ae0 aggregated-ether-options lacp active
```
This way, I created the ae interface and added the physical interfaces into that bundle.
Everything I configure on R1, I’ll repeat on the other routers according to the table.

Now we can configure all the interfaces like this:
```
set interfaces ge-0/0/2 unit 0 family inet address 10.200.0.2/31
set interfaces ge-0/0/3 unit 0 family inet address 10.200.0.4/31
set interfaces ge-0/0/3 unit 0 family inet6
set interfaces ae0 unit 0 family inet address 10.200.0.0/31
set interfaces ae0 unit 0 family inet6
set interfaces lo0 unit 0 family inet address 10.0.0.1/32
set interfaces lo0 unit 0 family inet6 address fd10:faca:f0fa::1/128
```
See that the IPv6 LLA configuration basically means enabling the inet6 family on the interface, nothing else is required for it to work.
You get the logic, right? Now you just need to configure the addresses on all routers following the same pattern.

Next, we need to configure VRRP to provide a redundant connection to our DC1!
This part isn’t so basic anymore, we’re going to configure VRRP with authentication and interface tracking to guarantee real redundancy.

But… why “real” redundancy?
With VRRP, you can run into a situation during the master election.
Imagine this: R4 loses all its backbone links, but it’s still technically alive and still communicating with R3 through DC1. Without tracking, R4 would stay as the master for that subnet, even though it’s completely isolated from the backbone.
With tracking, we can assign priority values to each interface.
If an interface goes down, the priority decreases.
If all backbone interfaces go down, the priority drops below the default master value 100.
This lets R3 take over as the master immediately.

You’ll see what I mean when we get to the configuration. Let’s go!

Let’s zoom into this scenario:

<img width="480" height="427" alt="image" src="https://github.com/user-attachments/assets/85c6880b-e7ae-4a6b-a3cd-aa88ce8d19bf" />

We'll use two VLANs with two different subnets, so I made a small table to guide the configuration.
| Router | Interface | VLAN | IP address | VIP | Priority | Authentication-type | Authentication-key |
| - | - | - | - | - | - | - | - |
| R3 | ge-0/0/0.110 | 110 | 172.30.110.1/24 | 172.30.110.254 | 150 | md5 | l4b |
| R3 | ge-0/0/0.111 | 111 | 172.30.111.1/24 | 172.30.111.254 | | md5 | l4b |
| R4 | ge-0/0/0.110 | 110 | 172.30.110.2/24 | 172.30.110.254 | | md5 | l4b |
| R4 | ge-0/0/0.111 | 111 | 172.30.111.2/24 | 172.30.111.254 | 150 | md5| l4b |

So, let's configure it!
R3:
```
set interfaces ge-0/0/0 unit 110 vlan-id 110
set interfaces ge-0/0/0 unit 110 family inet address 172.30.110.1/24 vrrp-group 1 virtual-address 172.30.110.254
set interfaces ge-0/0/0 unit 110 family inet address 172.30.110.1/24 vrrp-group 1 authentication-type md5
set interfaces ge-0/0/0 unit 110 family inet address 172.30.110.1/24 vrrp-group 1 authentication-key l4b
set interfaces ge-0/0/0 unit 111 vlan-id 111
set interfaces ge-0/0/0 unit 111 family inet address 172.30.111.1/24 vrrp-group 2 virtual-address 172.30.111.254
set interfaces ge-0/0/0 unit 111 family inet address 172.30.111.1/24 vrrp-group 2 priority 150
set interfaces ge-0/0/0 unit 111 family inet address 172.30.111.1/24 vrrp-group 2 authentication-type md5
set interfaces ge-0/0/0 unit 111 family inet address 172.30.111.1/24 vrrp-group 2 authentication-key l4b
```

R4:
```
set interfaces ge-0/0/0 unit 110 vlan-id 110
set interfaces ge-0/0/0 unit 110 family inet address 172.30.110.2/24 vrrp-group 1 virtual-address 172.30.110.254
set interfaces ge-0/0/0 unit 110 family inet address 172.30.110.2/24 vrrp-group 1 authentication-type md5
set interfaces ge-0/0/0 unit 110 family inet address 172.30.110.2/24 vrrp-group 1 authentication-key l4b
set interfaces ge-0/0/0 unit 111 vlan-id 111
set interfaces ge-0/0/0 unit 111 family inet address 172.30.111.2/24 vrrp-group 2 virtual-address 172.30.111.254
set interfaces ge-0/0/0 unit 111 family inet address 172.30.111.2/24 vrrp-group 2 priority 150
set interfaces ge-0/0/0 unit 111 family inet address 172.30.111.2/24 vrrp-group 2 authentication-type md5
set interfaces ge-0/0/0 unit 111 family inet address 172.30.111.2/24 vrrp-group 2 authentication-key l4b
```
This is enough to make it work... But remember the non-optimal master election issue.
We need to track some backbone interfaces to guarantee the best possible redundancy.

With tracking, we can assign priority values to the interfaces.
For example, on R3, VLAN 110 has priority 150, while the default VRRP priority is 100.
We can track the two backbone interfaces with a decrement of 30 each.

So if both backbone interfaces (the ones toward R2 and R6) go down, the total priority drops from 150 to 90, and then R4 becomes the master, which is exactly what we want.

Let’s configure that.
R3:
```
set interfaces ge-0/0/0 unit 110 family inet address 172.30.110.1/24 vrrp-group 1 track interface ge-0/0/3.0 priority-cost 30
set interfaces ge-0/0/0 unit 110 family inet address 172.30.110.1/24 vrrp-group 1 track interface ge-0/0/2.0 priority-cost 30
```
R4:
```
set interfaces ge-0/0/0 unit 111 family inet address 172.30.111.2/24 vrrp-group 2 track interface ge-0/0/3.0 priority-cost 30
set interfaces ge-0/0/0 unit 111 family inet address 172.30.111.2/24 vrrp-group 2 track interface ge-0/0/2.0 priority-cost 30
```

Now you get what I meant, I guess hahaha.
Let me disable the interface between R2 and R3.
<img width="731" height="801" alt="image" src="https://github.com/user-attachments/assets/044a0eb5-5f2c-4f25-9374-9f01c0826e09" />

```
root@R3# set interfaces ge-0/0/3 disable
root@R3# commit and-quit
kenv: unable to get vmtype
commit complete
Exiting configuration mode

root@R3> show vrrp track
Track Int   State         Speed   VRRP Int   Group   VR State      Current prio
ge-0/0/2.0  up               1g   ge-0/0/0.110     1 master                 120
ge-0/0/3.0  down             1g   ge-0/0/0.110     1 master                 120
```
Now you get what I meant — I think hahaha.
With one interface down, the priority value drops to 120.
And if we disable the other backbone interface, it will drop to 90.
Let’s check it out.
```
root@R3# set interfaces ge-0/0/2 disable

[edit]
root@R3# commit and-quit
kenv: unable to get vmtype
commit complete
Exiting configuration mode

root@R3> show vrrp track
Track Int   State         Speed   VRRP Int   Group   VR State      Current prio
ge-0/0/2.0  down             1g   ge-0/0/0.110     1 backup                  90
ge-0/0/3.0  down             1g   ge-0/0/0.110     1 backup                  90

root@R3> show vrrp
Interface     State       Group   VR state VR Mode   Timer    Type   Address
ge-0/0/0.110  up              1   backup   Active      D  3.274 lcl    172.30.110.1
                                                                vip    172.30.110.254
                                                                mas    172.30.110.2
ge-0/0/0.111  up              2   backup   Active      D  2.908 lcl    172.30.111.1
                                                                vip    172.30.111.254
                                                                mas    172.30.111.2
```
With the value at 90, the default priority 100 on R4 makes it become the master. Now it’s clear, right?

So, with this done, we can move on to the IGP configuration. But first, I’ll make some adjustments in our network.

See you later, my friend.



