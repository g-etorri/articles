# L3VPN Configuration

What's up fellas. Today we'll start to delivery some services to our customers!!! We'll start with L3VPNs. 

The topology you already... No!!! This time we have a new topology image! 
<img width="1146" height="822" alt="image" src="https://github.com/user-attachments/assets/d7e82fbe-fd3a-4abb-872e-a6ff15d2a601" />

First, let's made the boriest step of the topology. Configure all interfaces accordingly this table:
| Router | Interface    | IP Address    |  IPv6 Address             | Customer    |
| ------ | ------------ | ------------- | ------------------------- | ----------- |
| R1     | ge-0/0/8.200 | 10.2.1.1/30   |                           | C2-Hub      |
| R1     | ge-0/0/8.201 | 10.2.1.5/30   |                           | C2-Spoke    |
| R1     | ge-0/0/8.202 | 10.2.1.9/30   |                           | C2-Internet |
| R1     | lo0.1        | 10.2.1.254/32 |                           | C2-Hub      |
| R1     | lo0.2        | 10.2.1.253/32 |                           | C2-Spoke    |
| R2     | ge-0/0/7.200 | 10.2.2.1/30   |                           | C2-Hub      |
| R2     | ge-0/0/7.201 | 10.2.2.5/30   |                           | C2-Spoke    |
| R2     | ge-0/0/7.202 | 10.2.2.9/30   |                           | C2-Internet |
| R2     | lo0.1        | 10.2.2.254/32 |                           | C2-Hub      |
| R2     | lo0.2        | 10.2.2.253/32 |                           | C2-Spoke    |
| R3     | ge-0/0/1.300 |               | fc09:c0:ffee:3:3::1/126   | C3          |
| R3     | ge-0/0/8.100 | 10.1.3.1/30   |                           | C1          |
| R3     | lo0.1        | 10.1.3.254/32 |                           | C1          |
| R3     | lo0.2        | 10.3.3.254/32 | fc09:c0:ffee:3:3::254/128 | C3          |
| R4     | ge-0/0/8.100 | 10.1.4.1/30   |                           | C1          |
| R4     | ge-0/0/9.201 | 10.2.4.1/30   |                           | C2          |
| R4     | lo0.1        | 10.1.4.254/32 |                           | C1          |
| R4     | lo0.2        | 10.2.4.254/32 |                           | C2          |
| R5     | ge-0/0/7.201 | 10.2.5.1/30   |                           | C2          |
| R5     | lo0.1        | 10.2.5.254/32 |                           | C2          |
| R6     | ge-0/0/4.100 | 10.1.6.1/30   |                           | C1          |
| R6     | lo0.1        | 10.1.6.254/32 |                           | C1          |
| R7     | ge-0/0/6.201 | 10.2.7.1/30   |                           | C2          |
| R7     | lo0.1        | 10.2.7.254/32 |                           | C2          |
| R8     | ge-0/0/6.100 | 10.1.8.1/30   |                           | C1          |
| R8     | ge-0/0/8.300 |               | fc09:c0:ffee:3:8::1/126   | C3          |
| R8     | lo0.1        | 10.1.8.254/32 |                           | C1          |
| R8     | lo0.2        | 10.3.8.254/32 | fc09:c0:ffee:3:8::254/128 | C3          |

Here we're following a standard for the point-to-point networks. For the IPv4 network, the model is 10.X.Y.0/24, where X is the customer number and Y the router number, and the last octet is ordenated according to use. 
For the IPv6 networks, we'll use a similar model, fc09:c0:ffee:X:Y::/80, where X is the customer number and Y the router number, and the last octet is ordenated according to use. 

Ok, with this defined, let's make the configuration. I will use the R1 as example again, and we can apply similarly on the other routers. 
```
set interfaces ge-0/0/8 description to-C2-1
set interfaces ge-0/0/8 flexible-vlan-tagging
set interfaces ge-0/0/8 encapsulation flexible-ethernet-services
set interfaces ge-0/0/8 unit 200 description to-C2-HUB
set interfaces ge-0/0/8 unit 200 vlan-id 200
set interfaces ge-0/0/8 unit 200 family inet address 10.2.1.1/30
set interfaces ge-0/0/8 unit 201 description to-C2-SPOKE
set interfaces ge-0/0/8 unit 201 vlan-id 201
set interfaces ge-0/0/8 unit 201 family inet address 10.2.1.5/30
set interfaces ge-0/0/8 unit 202 description to-C2-INTERNET
set interfaces ge-0/0/8 unit 202 vlan-id 202
set interfaces ge-0/0/8 unit 202 family inet address 10.2.1.9/30
set interfaces lo0 unit 1 family inet address 10.2.1.254/32
set interfaces lo0 unit 2 family inet address 10.2.1.253/32
```
