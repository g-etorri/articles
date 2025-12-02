# Firewall Filter Configuration

Hey everyone! Today we’re going to set up some router protection by configuring a firewall filter for the RE. While this is standard stuff for JNCIA students, it’s absolutely essential for every network out there.

Let's build a filter to protect the RE that permits only the necessary protocols and allows for device management. This post will be pretty straightforward since we just need to create specific terms for each protocol we want to run.

I'll break down each term as we go:

In our lab, we're running ISIS. Since ISIS packets operate at Layer 2, we don't actually need to explicitly permit them in this specific filter.

However, we do need BFD to improve network convergence, allowing us to detect failures much faster. For this, we'll only permit port 3784. Port 3785 is used for BFD echo, which we don't need right now.
```
set firewall family inet filter filter-re term ALLOW-BFD from protocol udp
set firewall family inet filter filter-re term ALLOW-BFD from port 3784
set firewall family inet filter filter-re term ALLOW-BFD then accept
```

In our lab, we're using VRRP and OSPF to connect the access network to our Data Centers.
For the firewall filter, we can simply match on the "protocol". Just like OSPF, VRRP is encapsulated directly in IP packets and uses its own unique protocol number. We could create a term to match the specific multicast group to accept these packets, but I think matching by protocol is much simpler.
```
set firewall family inet filter filter-re term ALLOW-VRRP from protocol vrrp
set firewall family inet filter filter-re term ALLOW-VRRP then accept
set firewall family inet filter filter-re term ALLOW-OSPF from protocol ospf
set firewall family inet filter filter-re term ALLOW-OSPF then accept
```
For MPLS, we need to permit both LDP and RSVP. LDP is pretty simple, but man, I love RSVP! TE is just the coolest thing ever.
```
set firewall family inet filter filter-re term ALLOW-LDP from protocol udp
set firewall family inet filter filter-re term ALLOW-LDP from protocol tcp
set firewall family inet filter filter-re term ALLOW-LDP from port ldp
set firewall family inet filter filter-re term ALLOW-LDP then accept
set firewall family inet filter filter-re term ALLOW-RSVP from protocol rsvp
set firewall family inet filter filter-re term ALLOW-RSVP then accept
```
To ensure Multicast runs correctly, we need to permit PIM, IGMP, and MSDP.
PIM handles the heavy lifting and will be the vital function in our backbone. 
MSDP is responsible for discovering the source of a multicast stream.
Finally, we need IGMP so our customers can join a multicast group and receive the stream.
```
set firewall family inet filter filter-re term ALLOW-PIM from protocol pim
set firewall family inet filter filter-re term ALLOW-PIM then accept
set firewall family inet filter filter-re term ALLOW-MSDP from protocol tcp
set firewall family inet filter filter-re term ALLOW-MSDP from port msdp
set firewall family inet filter filter-re term ALLOW-MSDP then accept
set firewall family inet filter filter-re term ALLOW-IGMP from protocol igmp
set firewall family inet filter filter-re term ALLOW-IGMP then accept
```

Alright, time for BGP. Here's the move to keep our RE locked down:
You build a super slick prefix-list that automatically grabs all those dynamically configured peers. Then, toma, you just drop that list right into the firewall term. That’s how you keep your setup tight.
```
set policy-options prefix-list bgp-peers apply-path "protocols bgp group <*> neighbor <*>"
set firewall family inet filter filter-re term ALLOW-BGP from source-prefix-list bgp-peers
set firewall family inet filter filter-re term ALLOW-BGP from protocol tcp
set firewall family inet filter filter-re term ALLOW-BGP from port bgp
set firewall family inet filter filter-re term ALLOW-BGP then accept
```

For our general utility protocols, we’re keeping security maxed out. We only permit traffic originating from the management slice, which is the 10.71.0.0/24 network. Our required services? We’re allowing NTP, SNMP, RADIUS, DNS, FTP, and SSH. And for the sake of completion, we’ll let Telnet slide in, but we all know that one’s retired. 
```
set firewall family inet filter filter-re term ALLOW-NTP from source-address 10.71.0.0/24
set firewall family inet filter filter-re term ALLOW-NTP from protocol udp
set firewall family inet filter filter-re term ALLOW-NTP from port ntp
set firewall family inet filter filter-re term ALLOW-NTP then accept
set firewall family inet filter filter-re term ALLOW-SNMP from source-address 10.71.0.0/24
set firewall family inet filter filter-re term ALLOW-SNMP from protocol udp
set firewall family inet filter filter-re term ALLOW-SNMP from port snmp
set firewall family inet filter filter-re term ALLOW-SNMP then accept
set firewall family inet filter filter-re term ALLOW-RADIUS from source-address 10.71.0.0/24
set firewall family inet filter filter-re term ALLOW-RADIUS from protocol udp
set firewall family inet filter filter-re term ALLOW-RADIUS from port radius
set firewall family inet filter filter-re term ALLOW-RADIUS then accept
set firewall family inet filter filter-re term ALLOW-DNS from source-address 10.71.0.0/24
set firewall family inet filter filter-re term ALLOW-DNS from protocol udp
set firewall family inet filter filter-re term ALLOW-DNS from port domain
set firewall family inet filter filter-re term ALLOW-DNS then accept
set firewall family inet filter filter-re term ALLOW-SSH from source-address 10.71.0.0/24
set firewall family inet filter filter-re term ALLOW-SSH from protocol tcp
set firewall family inet filter filter-re term ALLOW-SSH from port ssh
set firewall family inet filter filter-re term ALLOW-SSH then accept
set firewall family inet filter filter-re term ALLOW-TELNET from source-address 10.71.0.0/24
set firewall family inet filter filter-re term ALLOW-TELNET from protocol tcp
set firewall family inet filter filter-re term ALLOW-TELNET from port telnet
set firewall family inet filter filter-re term ALLOW-TELNET then accept
set firewall family inet filter filter-re term ALLOW-FTP from source-address 10.71.0.0/24
set firewall family inet filter filter-re term ALLOW-FTP from protocol tcp
set firewall family inet filter filter-re term ALLOW-FTP from port ftp
set firewall family inet filter filter-re term ALLOW-FTP from port ftp-data
set firewall family inet filter filter-re term ALLOW-FTP then accept
```

To mitigate potential denial-of-service conditions, we will implement a policer to strictly rate-limit specific traffic types. We are limiting incoming ICMP and traceroute requests to 100 Kbps. This ensures the router can respond to diagnostic tools while preventing resource exhaustion.
```
set firewall policer 100k-on-rate if-exceeding bandwidth-limit 100k
set firewall policer 100k-on-rate if-exceeding burst-size-limit 25k
set firewall policer 100k-on-rate then discard
set firewall family inet filter filter-re term ALLOW-ICMP from protocol icmp
set firewall family inet filter filter-re term ALLOW-ICMP then policer 100k-on-rate
set firewall family inet filter filter-re term ALLOW-ICMP then accept
set firewall family inet filter filter-re term ALLOW-TRACEROUTE from protocol udp
set firewall family inet filter filter-re term ALLOW-TRACEROUTE from port 33434-33534
set firewall family inet filter filter-re term ALLOW-TRACEROUTE then policer 100k-on-rate
set firewall family inet filter filter-re term ALLOW-TRACEROUTE then accept
And let's make a counter to know how many packets are dropped by our filter. 
set firewall family inet filter filter-re term DROP-AND-COUNT then count dropped-packets
set firewall family inet filter filter-re term DROP-AND-COUNT then log
set firewall family inet filter filter-re term DROP-AND-COUNT then discard
```
Ready to test this thing? Let's prove our filter actually works! I’m going to change the ALLOW-SSH term from accept to reject right now to block all management access.
```
root@R1# show | compare
[edit firewall family inet filter filter-re term ALLOW-SSH then]
-       accept;
+       reject;

[edit]
root@R1# commit and-quit
kenv: unable to get vmtype
commit complete
Exiting configuration mode
[admin@MK-GW-1] > sys ssh user=noc address=10.71.0.1
connectHandler: Host is unreachable

Welcome back!
```
Boom! Connection refused! Our filter is definitely locked down!
Now, for the true confirmation, let’s quickly rollback that change to restore connectivity and make sure we can manage the device again.
```
root@R1# rollback 1
load complete

[edit]
root@R1# show |compare
[edit firewall family inet filter filter-re term ALLOW-SSH then]
-       reject;
+       accept;

[edit]
root@R1# commit and-quit
kenv: unable to get vmtype
commit complete
Exiting configuration mode

root@R1>

[admin@MK-GW-1] > sys ssh user=noc address=10.71.0.1
Password:
Last login: Tue Dec  2 13:28:18 2025 from 10.71.0.254
--- JUNOS 24.4R1.9 Kernel 64-bit  JNPR-15.0-20241104.1ed86e6_buil
noc@R1> exit


Welcome back!
```

This task is officially accomplished! Our RE is now secured with a robust firewall filter. 

The next step is to configure our interfaces and IP addresses. See you next time!
