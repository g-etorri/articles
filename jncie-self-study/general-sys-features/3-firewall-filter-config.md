# Firewall Filter Configuration

Hey everyone!
Today we’re going to set up some router protection by configuring a firewall filter for the RE. This is beginner-level stuff for anyone studying for the JNCIA, but it’s mandatory in any real network, no exceptions.

We’re going to build a filter to protect the Routing Engine, allowing only the protocols we actually need and permitting management access to the device. Nothing too complex here: we’ll create specific terms for each protocol we want to allow.

Let’s break things down term by term:

In our lab, we’re running IS-IS. Since IS-IS packets operate directly at Layer 2, so there’s no need to explicitly allow them in this filter.

But we do need BFD, which helps us improve convergence by detecting failures much faster. For that, we’ll allow only UDP port 3784, which is the asynchronous BFD control packets. Port 3785 is for BFD echo mode, and we’re not using that here, so there’s no need to permit it.
```
set firewall family inet filter filter-re term ALLOW-BFD from protocol udp
set firewall family inet filter filter-re term ALLOW-BFD from port 3784
set firewall family inet filter filter-re term ALLOW-BFD then accept
```

In our lab, we're using VRRP and OSPF to connect the access network to our Data Centers.
For the firewall filter, we can simply match on the protocol field. Just like OSPF, VRRP is encapsulated directly in IP packets and uses its own protocol number.
We could create a term to match the specific multicast group to allow these packets, but honestly, matching by protocol is much cleaner and easier to manage.
```
set firewall family inet filter filter-re term ALLOW-VRRP from protocol vrrp
set firewall family inet filter filter-re term ALLOW-VRRP then accept
set firewall family inet filter filter-re term ALLOW-OSPF from protocol ospf
set firewall family inet filter filter-re term ALLOW-OSPF then accept
```
For MPLS, we need to allow both LDP and RSVP.
LDP is straightforward, it just uses TCP/UDP port 646, nothing fancy.
RSVP, though… man, that’s my favorite. Traffic Engineering is just awesome. RSVP uses protocol number 46, so we can simply match on the protocol to permit it. Clean and simple.
```
set firewall family inet filter filter-re term ALLOW-LDP from protocol udp
set firewall family inet filter filter-re term ALLOW-LDP from protocol tcp
set firewall family inet filter filter-re term ALLOW-LDP from port ldp
set firewall family inet filter filter-re term ALLOW-LDP then accept
set firewall family inet filter filter-re term ALLOW-RSVP from protocol rsvp
set firewall family inet filter filter-re term ALLOW-RSVP then accept
```
To make sure multicast works smoothly, we need to allow PIM, IGMP, and MSDP through the firewall filter.

PIM does most of the heavy lifting and will be the key multicast protocol running in our backbone.
MSDP is what allows routers to exchange information about multicast sources across different domains.
And finally, IGMP is essential for the access side, it’s how clients join a multicast group and actually receive the stream.
```
set firewall family inet filter filter-re term ALLOW-PIM from protocol pim
set firewall family inet filter filter-re term ALLOW-PIM then accept
set firewall family inet filter filter-re term ALLOW-MSDP from protocol tcp
set firewall family inet filter filter-re term ALLOW-MSDP from port msdp
set firewall family inet filter filter-re term ALLOW-MSDP then accept
set firewall family inet filter filter-re term ALLOW-IGMP from protocol igmp
set firewall family inet filter filter-re term ALLOW-IGMP then accept
```

Alright, time for BGP. Here’s how we keep our RE locked down:
We create a clean prefix list that automatically matches every dynamically configured peer. Then, toma, we drop that prefix list straight into our firewall filter term. That’s how you keep the setup tight, secure, and zero-stress.
```
set policy-options prefix-list bgp-peers apply-path "protocols bgp group <*> neighbor <*>"
set firewall family inet filter filter-re term ALLOW-BGP from source-prefix-list bgp-peers
set firewall family inet filter filter-re term ALLOW-BGP from protocol tcp
set firewall family inet filter filter-re term ALLOW-BGP from port bgp
set firewall family inet filter filter-re term ALLOW-BGP then accept
```

For our general utility protocols, we’re keeping security maxed out. We’ll only permit traffic coming from the management slice, the 10.71.0.0/24 network.

As for the essential services, we’re allowing NTP, SNMP, RADIUS, DNS, FTP, and SSH.
And just for the sake of completeness, we’ll leave Telnet in there too… even though we all know that one is basically retired at this point.
```
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

We need to pay a little more attention to NTP.
The NTP term requires us to include the router’s own loopback address. Why? Because when you run “show ntp status” or “show ntp associations”, the router actually queries itself to fetch that information.

Yeah, that’s right — the box literally asks:
“Hey me, are you synchronized?”

If the loopback isn’t allowed in the filter, this check fails, and NTP looks broken even if everything else is fine.
(https://supportportal.juniper.net/s/article/Junos-Why-does-the-Network-Time-Protocol-NTP-stop-working-if-a-loopback-firewall-filter-is-applied)
```
root@R1> show configuration firewall family inet filter filter-re term ALLOW-NTP | display set
set firewall family inet filter filter-re term ALLOW-NTP from source-address 10.71.0.0/24
set firewall family inet filter filter-re term ALLOW-NTP from source-address 10.0.0.1/32
set firewall family inet filter filter-re term ALLOW-NTP from protocol udp
set firewall family inet filter filter-re term ALLOW-NTP from port ntp
set firewall family inet filter filter-re term ALLOW-NTP then accept
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
