# LDP Configuration

Hello guys! Today we'll configure LDP signaling in our network. Our goal is to create two LDP islands.

You already know the topology:

<img width="1014" height="747" alt="image" src="https://github.com/user-attachments/assets/266d8c1a-86f7-46cb-9602-323d9823b059" />

To make things more interesting, let's include some constraints:
* All LDP sessions must have MD5 authentication and track the IGP metrics.
* Our IGP has to sync with LDP.
* On R1 and R2, we need to advertise the IX interface as a FEC, but each prefix needs to have a different label.
* Our routers must perform ultimate-hop popping.

So, let's start configuring LDP on our backbone interfaces.

We need to add family mpls to our isis-if group. This way, we don't need to apply the family under every interface where ISIS is already running:
```
set groups isis-if interfaces <*> unit <*> family mpls
```

In the LDP configuration, let's add the required interfaces on each router according to the topology, and define MD5 authentication for the LDP neighbors. Here is an example for R1:
```
set protocols ldp interface ge-0/0/2.0
set protocols ldp interface ae0.0
set protocols ldp interface lo0.0
set protocols ldp session 10.0.0.2 authentication-key "$9$oOaUHP5Q3nC"
set protocols ldp session 10.0.0.4 authentication-key "$9$uDzA0IcevWXxd"

set protocols mpls interface ae0.0 
set protocols mpls interface ge-0/0/2.0 
```
We need to replicate this configuration across all routers in our network. Remember we are building LDP islands, so establish the sessions according to the topology.

Let's check our LDP neighbors:
```
root@R1> show ldp neighbor 
10.200.0.1                          ae0.0           10.0.0.2:0           11
10.200.0.3                          ge-0/0/2.0      10.0.0.4:0           13
```
Awesome, the LDP sessions are established. Let's verify R1's advertisements on R2:
```
root@R2> show ldp database session 10.0.0.1 
Input label database, 10.0.0.2:0--10.0.0.1:0
Labels received: 8
  Label     Prefix
      3      10.0.0.1/32
```
As you can see, R1 is advertising the implicit-null label. In other words, we are doing penultimate-hop popping. However, one of our goals is to guarantee that all routers do ultimate-hop popping!

To guarantee this, we need to advertise the explicit-null label:
```
set protocols ldp explicit-null
```
With this configuration, if everything goes as expected, we should receive the explicit-null label now:
```
root@R2> show ldp database session 10.0.0.1    
Input label database, 10.0.0.2:0--10.0.0.1:0
Labels received: 8
  Label     Prefix
      0      10.0.0.1/32
```
Looks good!

Now for another situation: on R1 and R2 specifically, we need to advertise the IX interfaces as a FEC. We need to create an egress policy and apply it to LDP to achieve this:
```
set policy-options policy-statement ldp-policy term 1 from protocol direct
set policy-options policy-statement ldp-policy term 1 from route-filter 10.0.0.1/32 exact
set policy-options policy-statement ldp-policy term 1 from route-filter 192.168.12.0/24 exact
set policy-options policy-statement ldp-policy term 1 then accept

set protocols ldp egress-policy ldp-policy
```
Now the routers are advertising this network as a FEC in our IGP, but we have a problem to handle. The routers are advertising this additional FEC with the exact same label. Check it out:
```
Input label database, 10.0.0.3:0--10.0.0.4:0
Labels received: 4
  Label     Prefix
 299776      10.0.0.1/32
 299792      10.0.0.2/32
 299808      10.0.0.3/32
      0      10.0.0.4/32
 299776      192.168.12.0/24

Output label database, 10.0.0.3:0--10.0.0.4:0
Labels advertised: 4
  Label     Prefix
 299792      10.0.0.1/32
 299776      10.0.0.2/32
      0      10.0.0.3/32
 299808      10.0.0.4/32
 299776      192.168.12.0/24
```
This is the output of the LDP session between R3 and R4. R4 is advertising the FECs from R1 using the same label, and R3 is doing the same.

To change this behavior, we need to configure LDP to deaggregate the FECs into distinct labels. We must apply this to all routers in our network to improve load balancing.
```
set protocols ldp deaggregate
```
Now, the routers will advertise each FEC with a different label:
```
Input label database, 10.0.0.3:0--10.0.0.4:0
Labels received: 5
  Label     Prefix
 299888      10.0.0.1/32
 299792      10.0.0.2/32
 299808      10.0.0.3/32
      0      10.0.0.4/32
 299776      192.168.12.0/24

Output label database, 10.0.0.3:0--10.0.0.4:0
Labels advertised: 5
  Label     Prefix
 299792      10.0.0.1/32
 299888      10.0.0.2/32
      0      10.0.0.3/32
 299808      10.0.0.4/32
 299776      192.168.12.0/24
```
Another goal achieved!

Now, let's sync LDP with the IGP. In IS-IS, we'll use ldp-synchronization to increment the interface metric when the LDP session is not fully established.
```
set protocols isis interface all ldp-synchronization
```
And in LDP, we'll configure it to track the IGP metrics.
```
set protocols ldp track-igp-metric
```

Let's check the results:
```
root@R1> show route table inet.3 protocol ldp 

inet.3: 15 destinations, 20 routes (7 active, 0 holddown, 11 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32        *[LDP/9] 1w4d 14:31:06, metric 5
                    >  to 10.200.0.1 via ae0.0, Push 0
10.0.0.3/32        *[LDP/9] 1w4d 14:26:10, metric 15
                    >  to 10.200.0.1 via ae0.0, Push 45
10.0.0.4/32        *[LDP/9] 1w4d 14:27:15, metric 10
                    >  to 10.200.0.3 via ge-0/0/2.0, Push 0
```  
LDP is successfully tracking the IGP metrics, but let's check the LDP sync now. I'll disable the LDP session between R1 and R2. The interface metric will jump to its maximum value, and the route should shift via R4 if everything works as expected.
```
root@R1> show isis interface ae0.0 detail    
IS-IS interface database:
ae0.0
  Index: 325, State: 0x6, Circuit id: 0x1, Circuit type: 1
  LSP interval: 100 ms, CSNP interval: 20 s
  Adjacency advertisement: Advertise, Layer2-map: Disabled
  Level Adjacencies Priority   Metric Hello (s) Hold (s) Designated Router
    1             1       64        5     9.000       27
root@R1> show route 10.0.0.2    

inet.0: 221 destinations, 243 routes (218 active, 0 holddown, 3 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32        *[IS-IS/15] 1w4d 14:32:48, metric 5
                    >  to 10.200.0.1 via ae0.0
...
deactivate protocols ldp session 10.0.0.2
...
root@R1> show isis interface ae0.0 detail 
IS-IS interface database:
ae0.0
  Index: 325, State: 0x6, Circuit id: 0x1, Circuit type: 1
  LSP interval: 100 ms, CSNP interval: 20 s
  Adjacency advertisement: Advertise, Layer2-map: Disabled
  Level Adjacencies Priority   Metric Hello (s) Hold (s) Designated Router
    1             1       64 16777214     9.000       27
root@R1> show route 10.0.0.2 

inet.0: 221 destinations, 243 routes (218 active, 0 holddown, 3 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32        *[IS-IS/15] 00:00:40, metric 30
                    >  to 10.200.0.3 via ge-0/0/2.0
```
Everything looks perfect! This wraps up our LDP configuration, but we don't have full MPLS connectivity yet.

In the next chapter, we'll configure RSVP LSPs to interconnect our LDP islands! See you soon.

