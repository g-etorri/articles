# LDP Configuration

Hello guys, today we'll configure the LDP signaling in our network. Our goal is make two LDP islands in our network. 

The topology, you already know:

<img width="1014" height="747" alt="image" src="https://github.com/user-attachments/assets/266d8c1a-86f7-46cb-9602-323d9823b059" />

To learn something more, we'll include some constraints:
* All LDP sessions must have MD5 authentication and track the IGP metrics. 
* Our IGP have to sync with LDP. 
* In the R1 and R2, we have to advertise the IX interface as a FEC, but each prefix need to have a different label. 
* Our routers must pop the labels by the ultimate router. 

So, let's start configuring the LDP in our backbone interfaces. 

We need to add the family mpls in our isis-if group, this way don't need to apply the family in each interface that we already have ISIS running. 
```
set groups isis-if interfaces <*> unit <*> family mpls
```

In LDP configuration, we let's add the interfaces of each router need, accordingly the topology and define the MD5 authentication with the LDP neighbors. Here I let an example of R1:
```
set protocols ldp interface ge-0/0/2.0
set protocols ldp interface ae0.0
set protocols ldp interface lo0.0
set protocols ldp session 10.0.0.2 authentication-key "$9$oOaUHP5Q3nC"
set protocols ldp session 10.0.0.4 authentication-key "$9$uDzA0IcevWXxd"

set protocols mpls interface ae0.0 
set protocols mpls interface ge-0/0/2.0 
```
We need to replicate this configuration in all router of our network, remember the LDP islands, so establish the sessions accordingly the topology. 

Let's check our LDP neighbors:
```
root@R1> show ldp neighbor 
10.200.0.1                          ae0.0           10.0.0.2:0           11
10.200.0.3                          ge-0/0/2.0      10.0.0.4:0           13
```
Ok, the LDP sessions are established. Let's verify the R1 advertisements on R2:
```
root@R2> show ldp database session 10.0.0.1 
Input label database, 10.0.0.2:0--10.0.0.1:0
Labels received: 8
  Label     Prefix
      3      10.0.0.1/32
```
As you can see, the R1 are advertising the implicit-null label, in other words, we are making the penultimate-hop-popping, and one of our goals is guarantee that all the routers do the ultimate-hop-popping. 

To guarantee that our routes will pop the label. We need to advertise the explicit-null label:
```
set protocols ldp explicit-null
```
With this configuration, if all occurs as expected, we'll receive the explicit-null label now:
```
root@R2> show ldp database session 10.0.0.1    
Input label database, 10.0.0.2:0--10.0.0.1:0
Labels received: 8
  Label     Prefix
      0      10.0.0.1/32
```
This is ok! 

Now, another situation, un R1 and R2 specifically, we need to advertise the IX interfaces as a FEC, we need to make a egress-policy and apply into the LDP to achieve this: 
```
set policy-options policy-statement ldp-policy term 1 from protocol direct
set policy-options policy-statement ldp-policy term 1 from route-filter 10.0.0.1/32 exact
set policy-options policy-statement ldp-policy term 1 from route-filter 192.168.12.0/24 exact
set policy-options policy-statement ldp-policy term 1 then accept

set protocols ldp egress-policy ldp-policy
```
With this, the routers are advertising this network as a FEC in our IGP, but we have a problem that we need to handle. The routers are advertising this additional FEC with the same label, see below:
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
This is a output of the LDP session between the R3 and R4, R4 is advertising the FECs of R1 with the same label, and R3 is doing the same.

To change this behavior, we need to set the LDP to deaggregate the FECs in labels. We must apply this in all routers of our network, with this we'll improve the load-balance of the network. 
```
set protocols ldp deaggregate
```
With this, the routers will advertise each FEC with a different label:
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
Another goal achieved!!!

Now, let's sync the LDP with the IGP:
In the ISIS, we'll use the ldp-synchronization to increment the metric of interfaces when the LDP sessions is not fully established. 
```
set protocols isis interface all ldp-synchronization
```
And in the LDP, we'll configure to track the ISIS metrics. 
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
The LDP is tracking the IGP metrics, but let's check the LDP sync now. I'll disable the LDP session between R1 and R2, the route should change via R4 if all occurs as expected. 
```
root@R1> show route 10.0.0.2    

inet.0: 221 destinations, 243 routes (218 active, 0 holddown, 3 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32        *[IS-IS/15] 1w4d 14:32:48, metric 5
                    >  to 10.200.0.1 via ae0.0
...
deactivate protocols ldp session 10.0.0.2
...
root@R1> show route 10.0.0.2 

inet.0: 221 destinations, 243 routes (218 active, 0 holddown, 3 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32        *[IS-IS/15] 00:00:40, metric 30
                    >  to 10.200.0.3 via ge-0/0/2.0
```
Everything looks perfect. This is our LDP configuration, but, we don't have full MPLS connectivity yet. 

In the next chapter, we'll configure RSVP LDPs to interconnect our LDP islands!!! See you soon. 

