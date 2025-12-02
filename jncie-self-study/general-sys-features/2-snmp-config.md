# SNMP Configuration

Hello everyone, 

This time we need to configure SNMP on our routers, so we can receive and collect information from them.
For the SNMPv3 visualization, we will follow the parameters below:
| Parameters | Value |
| ---------- | ----- |
| VACM user | lab |
| VACM security model | usm |
| VACM security level | privacy | 
| VACM read view OID | .1 |
| USM username | lab |
| USM user authentication | SHA |
| USM authentication password | L4B-SNMP-KEY |
| USM user encryption | 3DES | 
| USM user encryption password | L4B-PRIV-KEY

USM stands for User-based Security Model, and it defines how SNMPv3 authenticates and encrypts the communication between the server and the agent.
In simple terms, we will create a user that will be responsible for the secure connection between the SNMP manager and the device.

VACM stands for View-based Access Control Model, and it controls what the user is allowed to see inside the MIBs. In other words, it defines what the user can read or do.

First, let's configure the user along with the authentication and encryption keys according to our table:
```
set snmp v3 usm local-engine user lab authentication-sha authentication-password workbook
set local-engine user lab privacy-3des privacy-password workbook
```

Now we need to configure the MIB view.
MIBs follow a hierarchical OID structure, and we want full visibility of this hierarchy.

So, let’s configure the OID .1 to allow complete access:
```
set view root-view oid .1 include
```

MIB stands for Management Information Base, which is basically a big database of information inside the device.
Each MIB focuses on some specific part of the equipment, one handles interfaces, another handles system info, another handles routing, and so on.

Inside these MIBs we have the famous OIDs or Object Identifiers.
Think of an OID like an address that points exactly to the thing you want to monitor. Every single item has its own unique OID.

All OIDs follow a huge hierarchical tree, and everything starts from the root: 1.
From there, each branch adds more numbers, and that's how you get those long OID chains we use in SNMP.

With the diagram below, the hierarchy becomes way easier to visualize.
<img width="1473" height="686" alt="image" src="https://github.com/user-attachments/assets/376dd42b-4d97-4997-9349-4ad822036256" />

Now, let's configure the access control for our SNMP user as read-only:
```
set snmp v3 vacm access group primary-group default-context-prefix security-model usm security-level privacy read-view root-view
set snmp v3 vacm security-to-group security-model usm security-name lab group primary-group
```
The group name is primary-group, and inside it we define the security level (privacy) and the access level (read-only).
The root-view is simply the view identifier we created earlier, where we define the OID hierarchy that the user is allowed to access.
After that, we just need to assign the user to the group.

After that, we need to configure the SNMPv3 notification parameters 
Let's follow the parameters below:
| Parameters | Values |
| ---------- | ------ |
| Target Address | 10.71.0.253 |
| Target SNMP model | v3 |
| Target security model | usm |
| Target security level | privacy |
| Target security name | lab |
| Notifcation OID filter | snmpTraps, jnxTraps | 
| Notification type | trap |

First, we need to configure what we’ll notify via SNMP. In this case, it will be trap notifications.
```
set snmp v3 notify traps type trap
set snmp v3 notify traps tag tag-to-srv1
```
Here, traps is just an identifier, like a group name. Inside it, we define the notification type—in this case, traps.
The tag acts as an identifier to mark which notifications will be sent to the server.

Now, we can define the OIDs we want to send as traps:
```
set snmp v3 notify-filter all oid snmpTraps
set snmp v3 notify-filter all oid jnxTraps
```
These OIDs don’t show up directly in the CLI, so we need to consult the documentation to specify them correctly.

Next step is to configure the communication parameters with our SRV-1, the greatest!
The message-processing model will be SNMPv3, the security model will be USM, the security level will be privacy, and the security name will be lab. 
```
set snmp v3 target-parameters SRV-1-parameters parameters message-processing-model v3
set snmp v3 target-parameters SRV-1-parameters parameters security-model usm
set snmp v3 target-parameters SRV-1-parameters parameters security-level privacy
set snmp v3 target-parameters SRV-1-parameters parameters security-name lab
```
All of these parameters are mandatory, if you miss any of them, Junos will throw an error before the commit.

Finally, we need to configure the target IP address, the notification that will be sent with the tag, and the parameters defined earlier.
```
set snmp v3 target-address SRV-1 address 10.10.1.253
set snmp v3 target-address SRV-1 tag-list tag-to-srv1
set snmp v3 target-address SRV-1 target-parameters SRV-1-parameters
```
So, it's done.
Let’s check if this is working:
```
root@R1> show snmp v3

Local engine ID: 80 00 0a 4c 01 0a 00 00 01
Engine boots:           1
Engine time:       368450 seconds
Max msg size:       65507 bytes

Engine ID: 80 00 0a 4c 01 0a 47 00 01
    User                            Auth/Priv   Storage      Status
    lab                              sha/aes128 nonvolatile  active

Group name           Security  Security              Storage      Status
                     model     name                  type
primary-group        usm       lab                   nonvolatile  active

Access control:
Group                Context Security      Read       Write     Notify
                     prefix  model/level   view       view      view
primary-group                 usm/privacy  root-view

SNMP Target:
Address     Address                     Port  Parameters  Storage     Status
name                                          name        type
SRV-1       10.71.0.253                 162   SRV-1-param nonvolatile active

Parameters     Security        Security     Notify  Storage      Status
name           name            model/level  filter  type
SRV-1-paramete lab              usm/privacy         nonvolatile  active

SNMP Notify:
Notify               Tag                Type         Storage      Status
name                                                 type
traps                tag-to-srv1        trap         nonvolatile  active

Filter               Subtree            Filter       Storage      Status
name                                    type         type
all                  1.3.6.1.4.1.2636.  include      nonvolatile  active
all                  1.3.6.1.6.3.1.1.5  include      nonvolatile  active
```
In this output we can check our configuration. 

And let's run an SNMP walk to check the hostname!
```
root@kvm:/etc# snmpwalk -v3 -l authPriv -u lab -a SHA -A workbook -x AES -X workbook 10.10.1.1 1.3.6.1.2.1.1.5.0
iso.3.6.1.2.1.1.5.0 = STRING: "R1"
```

And we have success!!!

This is the second part of our JNCIE-SP journey. As the configuration gets more refined, things will start to get even more interesting.
Winter is coming...
