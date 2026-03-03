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
