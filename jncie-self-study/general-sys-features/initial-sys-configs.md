# Initial System Configurations

Hello everyone, 

Alright, so for the first post of this journey we’re starting right at the very beginning hahah.
Today we’re just doing the basic system setup on our routers.
Here’s the topology we’ll be working with:
<img width="1154" height="836" alt="image" src="https://github.com/user-attachments/assets/c6c625e8-7f5e-4e3c-b0a6-10807b798cbe" />
For some context, we have an SRV-1 acting as our DNS, NTP, FTP, and RADIUS server. It can also do SNMP gets and receive flow data.
We also have a Mikrotik router (a must-have for us Brazilians hahah) to connect the SRV-1 to the internet, and it can act as an SSH client for some tests as well.

First things first: we need to give our routers their hostnames and set up the mgmt interfaces.
Here’s the plan:
| Hostname | IP address |
| -------- | ---------- |
| R1 | 10.71.0.1/24 |
| R2 | 10.71.0.2/24 |
| R3 | 10.71.0.3/24 |
| R4 | 10.71.0.4/24 | 
| R5 | 10.71.0.5/24 |
| R6 | 10.71.0.6/24 |
| R7 | 10.71.0.7/24 |
| R8 | 10.71.0.8/24 |

If you’re reading this, I’m pretty sure you already know how to do this — but just in case you don’t:
```
set system host-name R1
set interfaces fxp0 description MGMT-OOB
set interfaces fxp0 unit 0 family inet address 10.71.0.1/24
```
And that’s it!
We’ll just repeat the same steps on all the other routers from here on out.

Let's ping everyone just to make sure things are alive, and we should see the ARP entries show up as well:
```
[admin@MK-GW-1] > ip arp pr
Flags: D, P - PUBLISHED; C - COMPLETE
Columns: ADDRESS, MAC-ADDRESS, INTERFACE
 #    ADDRESS       MAC-ADDRESS        INTERFACE
 0 DC 10.71.0.253   50:E0:60:00:03:00  vlan71-mgmt
 1 DC 10.71.0.1     50:3E:99:00:05:00  vlan71-mgmt
 3 DC 10.71.0.2     50:5D:F1:00:04:00  vlan71-mgmt
 5 DC 10.71.0.8     50:1E:D0:00:10:00  vlan71-mgmt
 6 DC 10.71.0.5     50:4C:C7:00:0D:00  vlan71-mgmt
 7 DC 10.71.0.7     50:A1:27:00:11:00  vlan71-mgmt
 8 DC 10.71.0.4     50:88:17:00:08:00  vlan71-mgmt
 9 DC 10.71.0.6     50:4D:78:00:0C:00  vlan71-mgmt
10 DC 10.71.0.3     50:DB:B4:00:09:00  vlan71-mgmt
```
Voilà!

Next up, we need to enable a couple of services on our routers: SSH, FTP, and Telnet. SSH and Telnet are for management, and FTP is just to make our life easier when moving files around between routers.
```
set system services ssh
set system services ftp
set system services telnet
```
IRL, I recommend using only SSH. You should already know about the security issues with Telnet, so keep it only for lab/testing purposes.
Let's check if our configuration are working: 
```
root@R1> show system connections | match LISTEN | match .2
tcp6       0      0  *.22                                          *.*                                           LISTEN
tcp4       0      0  *.22                                          *.*                                           LISTEN
tcp6       0      0  *.21                                          *.*                                           LISTEN
tcp4       0      0  *.21                                          *.*                                           LISTEN
tcp6       0      0  *.23                                          *.*                                           LISTEN
tcp4       0      0  *.23                                          *.*                                           LISTEN
```

Everything looks good, but let’s actually try to connect:
```
[admin@MK-GW-1] > sys ssh user=root address=10.71.0.1
Password:
Password:
Password:

Welcome back!
```
So, we’ve got two pieces of news here: one good and one not-so-good.
The good news is that the router is listening on TCP port 22.
The not-so-good news is… we can’t log in. Why?

Because we still need to enable a specific knob that allows root login over SSH.
Let’s fix that:
```
set system services ssh root-login allow
```
Let’s try again:
```
[admin@MK-GW-1] > sys ssh user=root address=10.71.0.1
Password:
Last login: Mon Nov 24 14:48:20 2025
--- JUNOS 24.4R1.9 Kernel 64-bit  JNPR-15.0-20241104.1ed86e6_buil
root@R1:~ #
```

Yayyy, now it works!!! 

