# RSVP Protection Configuration

Hello guys, today we'll configure protection in our RSVP LSPs!!!

You already know the topology:
<img width="1026" height="793" alt="image" src="https://github.com/user-attachments/assets/e0e9790b-3451-461a-8ef7-fa01e680a242" />

This time, I'll list all the tasks that we have to accomplish, like the JNCIE-SP study book. 
* Configure a backup path in all LSP except in R8-R3-B, R3-R8-B, R7-R4-B and R4-R7-B.
* Make sure that LSPs R1-R6-A, R6-R1-A, R2-R5-A and R5-R2-A have a backup path established in advance, before the primary fails.
* Configure all LSPs to not double the bandwidth with the secondary path.
* Configure the LSPs R2-R7-A, R7-R2-A, R3-R6-A and R6-R3-A to not come back to the primary path if fails.
* Configure the LSPs R1-R6-A, R6-R1-A, R2-R5-A and R5-R2-A to use fast-reroute, without use bandwidth or admin-group of the primary path. The devour path do not must have more than 5 hops.
* Configure the LSPs R1-R8-A, R8-R1-A, R2-R7-A, R7-R2-A, R3-R6-A, R6-R3-A, R4-R5-A and R5-R4-A to use link-protection mechanism.
* Configure the LSPs R8-R3-A, R3-R8-A, R7-R4-A and R4-R7-A to use the node-link-protection mechanism.

