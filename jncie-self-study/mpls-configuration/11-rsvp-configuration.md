# RSVP Configuration

Hello guys, today we'll connect our LDP islands using RSVP. And we'll make some TE preparations on our LAB!!! Let's go. 

You already know the topology:
<img width="1026" height="793" alt="image" src="https://github.com/user-attachments/assets/e0e9790b-3451-461a-8ef7-fa01e680a242" />

Note that we have the links colored, this is our admin-groups! 

Today, we have so much constraints! Because, making traffic engineering IRL we'll see some constraints, and we need to learn this to do IRL. 
First, we need to configure our backbone interfaces with RSVP. In this task, we need to accomplish the follow parameters:
* Configure the MD5 authentication in all backbone interfaces. 
* Configure the bandwidth of 333 Mbps in all interface, execpt the LAG interfaces.

This is simple, here we have the configuration in R1, and you can apply the configuration similarly in the other routers: 
```
set protocols rsvp interface ae0.0 authentication-key l4b
set protocols rsvp interface ge-0/0/2.0 authentication-key l4b
set protocols rsvp interface ge-0/0/2.0 bandwidth 333m
set protocols rsvp interface ge-0/0/3.0 authentication-key l4b
set protocols rsvp interface ge-0/0/3.0 bandwidth 333m
```
We can see that hello are exchangeds between R1 and R2:
```
root@R1> show rsvp interface ae0.0 detail | match Hello 
  HelloInterval 9(second)
  Hello                4             2           2             1
```

Ok, with the RSVP configured in all interfaces, let's color the interfaces, using admin-groups! We'll do it following the table below:
| Router | Interface  | Admin Group  |
| ------ | ---------- | ------------ |
| R1| ge-0/0/2.0 | blue         |
| R1| ge-0/0/3.0 | orange       |
| R1| ae0.0      | blue, orange |
| R2| ge-0/0/2.0 | blue         |
| R2| ge-0/0/2.0 | orange       |
| R2| ae0.0      | blue, orange |
| R3| ge-0/0/2.0 | blue         |
| R3| ge-0/0/3.0 | orange       |
| R3| ge-0/0/4.0 | blue, orange |
| R4| ge-0/0/2.0 | blue         |
| R4| ge-0/0/3.0 | orange       |
| R4| ge-0/0/4.0 | blue, orange |
| R5| ge-0/0/2.0 | blue         |
| R5| ge-0/0/3.0 | orange       |
| R5| ae0.0      | blue, orange |
| R6| ge-0/0/2.0 | blue         |
| R6| ge-0/0/3.0 | orange       |
| R6| ae0.0      | blue. orange |
| R7| ge-0/0/2.0 | blue         |
| R7| ge-0/0/3.0 | orange       |
| R7| ge-0/0/4.0 | blue. orange |
| R8| ge-0/0/2.0 | blue         |
| R8| ge-0/0/3.0 | orange       |
| R8| ge-0/0/4.0 | blue.orange  |

Again, I'll do this in R1 and apply similarly in the other routers of our network:
```
set protocols mpls admin-groups orange 0
set protocols mpls admin-groups blue 1
set protocols mpls interface ae0.0 admin-group orange
set protocols mpls interface ae0.0 admin-group blue
set protocols mpls interface ge-0/0/3.0 admin-group orange
set protocols mpls interface ge-0/0/2.0 admin-group blue
```

Now, with our underlay configured, we can configure our LSPs. Here we have a list of all LSPs that we need to configure in our network:
| Ingress | Egress | LSP Name |
| ------- | ------ | -------- |
| R1      | R8     | R1-R8-A  |
| R1      | R6     | R1-R6-A  |
| R2      | R7     | R2-R7-A  |
| R2      | R5     | R2-R5-A  |
| R3      | R8     | R3-R8-A  |
| R3      | R8     | R3-R8-B  |
| R3      | R6     | R3-R6-A  |
| R4      | R7     | R4-R7-A  |
| R4      | R7     | R4-R7-B  |
| R4      | R5     | R4-R5-A  |
| R5      | R2     | R5-R2-A  |
| R5      | R4     | R5-R4-A  |
| R6      | R1     | R6-R1-A  |
| R6      | R3     | R6-R3-A  |
| R7      | R2     | R7-R2-A  |
| R7      | R4     | R7-R4-A  |
| R7      | R4     | R7-R4-B  |
| R8      | R1     | R8-R1-A  |
| R8      | R3     | R8-R3-A  |
| R8      | R3     | R8-R3-B  |

First, let's make the standard configuration of the LSPs in our network. All the LSPs will have BFD working, and priority and hold with the worse value. 
To BFD working accordingly, we need to add a term in our RE filter, and modify another. 
```
set firewall family inet filter filter-re term ALLOW-BFD from port 4784
set firewall family inet filter filter-re term ALLOW-LSP-PING from protocol udp
set firewall family inet filter filter-re term ALLOW-LSP-PING from port 3503
set firewall family inet filter filter-re term ALLOW-LSP-PING then accept
edit firewall family inet filter filter-re
insert term ALLOW-LSP-PING before term DROP-AND-COUNT
```
We need to add the control port for BFD, before this we was using only the echo port. The BFD depends of the LSP ping also, so, we need to permit this in the RE filter. 
```
set protocols mpls label-switched-path R1-R8-A to 10.0.0.8
set protocols mpls label-switched-path R1-R8-A priority 7 7
set protocols mpls label-switched-path R1-R8-A oam bfd-liveness-detection minimum-interval 3000
set protocols mpls label-switched-path R1-R6-A to 10.0.0.6
set protocols mpls label-switched-path R1-R6-A priority 7 7
set protocols mpls label-switched-path R1-R6-A oam bfd-liveness-detection minimum-interval 3000
```
Note, by default the hold value on Junos will be 0, with this, even if we have a LSP with a best priority value, the preemption will not occur because the hold value is 0. We need to guarantee that our hold value is the worse, that's why the hold value need to be changed. 

With this, we'll have the LSPs and BFD sessions established! 
```
root@R1> show mpls lsp ingress 
Ingress LSP: 2 sessions
To              From            State Rt P     ActivePath       LSPname
10.0.0.6        10.0.0.1        Up     0 *                      R1-R6-A
10.0.0.8        10.0.0.1        Up     0 *                      R1-R8-A
Total 2 displayed, Up 2, Down 0

root@R1> show bfd session 
                                                  Detect   Transmit
Address                  State     Interface      Time     Interval  Multiplier
10.0.0.6                 Up                       9.000     3.000        3   
10.0.0.8                 Up                       9.000     3.000        3  
```

Now, let's define the admin-groups of each LSP, half of the LSPs wil go by blue, and another half go by orange: 
```
set protocols mpls label-switched-path R2-R7-A admin-group include-any orange
set protocols mpls label-switched-path R3-R6-A admin-group include-any orange
set protocols mpls label-switched-path R6-R3-A admin-group include-any orange
set protocols mpls label-switched-path R7-R2-A admin-group include-any orange
set protocols mpls label-switched-path R1-R8-A admin-group include-any blue
set protocols mpls label-switched-path R8-R1-A admin-group include-any blue
set protocols mpls label-switched-path R4-R5-A admin-group include-any blue
set protocols mpls label-switched-path R5-R4-A admin-group include-any blue
```

