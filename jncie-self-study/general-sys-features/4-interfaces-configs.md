# Interfaces Configuration

Hey everyone, today we'll configure the addresses of our backbone interfaces, and the interface to connect our DC1!!!
To don't waste time, here we go. 
We'll use the topoogy below:
<img width="1211" height="747" alt="image" src="https://github.com/user-attachments/assets/51ef5469-08d4-4bed-a2d8-81fb1244a531" />

And we'll follow the parameters below:
| Router | Interface | IPv4 address | IPv6 address |
| ------ | --------- | ------------ | ------------ |
| R1 | ae0.0 | 10.200.0.0/31 | link-local |
| R1 | ge-0/0/2.0 | 10.200.0.2/31 | |
| R1 | ge-0/0/3.0 | 10.200.0.4/31 | |
| R2 | ae0.0 | 10.200.0.1/31 | link-local | 
| R2 | ge-0/0/3.0 | 10.200.0.6/31 | link-local |
| R2 | ge-0/0/2.0 | 10.200.0.8/31 | | 
| R3 | ge-0/0/3.0 | 10.200.0.7/31 | link-local |
| R3 | ge-0/0/4.0 | 10.200.0.10/31 | link-local |
| R3 | ge-0/0/2.0 | 1.0200.0.12/31 | |
| R4 | ge-0/0/4.0 | 10.200.0.11/31 | link-local |
| R4 | ge-0/0/3.0 | 10.200.0.14/31 | link-local | 
| R4 | ge-0/0/2.0 | 10.200.0.3/31 | |
| R5 | ge-0/0/3.0 | 10.200.0.15/31 | link-local |
| R5 | ae0.0 | 10.200.0.16/31 | link-local | 
| R5 | ge-0/0/2.0 | 10.200.0.18/31 | |
| R6 | ae0.0 | 10.200.0.17/31 | link-local |
| R6 | ge-0/0/2.0 | 10.200.0.13/31 | |
| R6 | ge-0/0/3.0 | 10.200.0.20/31 | link-local |
| R7 | ge-0/0/3.0 | 10.200.0.21/31 | link-local |
| R7 | ge-0/0/2.0 | 10.200.0.9/31 | |
| R7 | ge-0/0/4.0 | 10.200.0.22/31 | link-local |
| R8 | ge-0/0/4.0 | 10.200.0.23/31 | link-local |
| R8 | ge-0/0/3.0 | 10.200.0.5/31 | link-local |
| R8 | ge-0/0/2.0 | 10.200.0.19/31 | |

Ok, the first thing to do is create de LACP interfaces, A.K.A. "ae" or aggregated ethernet interfaces. 
So, we need to do this in the links of R1 to R2 and R5 to R6, let's go. 
