# SR-MPLS Configuration

Hello guys! After mastering RSVP protection, it's time to dive into SR-MPLS. SR is a game-changer because it simplifies the control plane by removing the need for LDP/RSVP, using the IGP (ISIS or OSPF) to distribute labels.

The topology you already know:
<img width="1033" height="783" alt="image" src="https://github.com/user-attachments/assets/7ab718e7-c78f-420c-8b23-978f8b04e62d" />

Now, follow the same style of the previous post, let's list our tasks:
* Configure SR-MPLS in all routers, defining a block of 5000 labels and ensuring that the final label was 15000. Using 100+ Router number as Segment-ID of the router.
* Ensure that the R1 allocate the labels statically between 40000 and 45000.
* In R5, make a static SR-TE to R1 that passes trough R8. In R1, made a SR-TE to R5 that passes trough R7. Don't specific the SIDs explicitly in the path.
* In R2, made a static SR-TE to R5 that passes trough R4. Without specificating the SIDs explicitly in the path. In R5, made a static SR-TE to R2, passing trough R1 and arriving in R2 trough the interface ge-0/0/1 of the LAG. 
* In all routers of the network, enable TI-LFA link and node-protection for the routes in the inet.0 and inet.3.
* Configure LDP tunneling in the SR-TEs between R1 and R5, and R2 and R5.
* In R3, confiture a dynamic SR-TE to R4 that passes trough R2. Ensure that this dynamic path only be used if don't have LDP routes to R4.
* In R3, configure the microloop avoidance, and ensure that the convergence paths will be removed after 10 seconds.

