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

Toma (BR moment), now it works!!! 

Now we’re going to configure the DNS and NTP servers on all our routers.
We also need to set the correct time zone. For the NTP server, let’s use MD5 authentication, and the key will be l4b-ntp:
```
set system name-server 10.71.0.253
set system time-zone America/Sao_Paulo
set system ntp authentication-key 1 type md5
set system ntp authentication-key 1 value l4b-ntp
set system ntp server 10.71.0.253 key 1
set system ntp trusted-key 1
```
Let’s check if everything is working.

For DNS, I created a simple list of domains based on the routers’ interface names just to make life easier.
So let’s ping r2.fxp0.jncie.lab to confirm DNS resolution:
```
root@R1> ping r2.fxp0.jncie.lab
PING r2.fxp0.jncie.lab (10.71.0.2): 56 data bytes
64 bytes from 10.71.0.2: icmp_seq=0 ttl=64 time=0.588 ms
64 bytes from 10.71.0.2: icmp_seq=1 ttl=64 time=0.807 ms
64 bytes from 10.71.0.2: icmp_seq=2 ttl=64 time=0.774 ms
64 bytes from 10.71.0.2: icmp_seq=3 ttl=64 time=0.707 ms
64 bytes from 10.71.0.2: icmp_seq=4 ttl=64 time=0.761 ms
^C
--- r2.fxp0.jncie.lab ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.588/0.727/0.807/0.077 ms
```
Perfect, it's working! 
Now let’s check the NTP server:
```
root@R1> show ntp status
associd=0 status=0615 leap_none, sync_ntp, 1 event, clock_sync,
version="ntpd 4.2.8p15-a Thu Dec 19 07:58:36  2024 (1)",
processor="amd64", system="FreeBSDJNPR-15.0-20241104.1ed86e6_buil",
leap=00, stratum=4, precision=-24, rootdelay=13.442, rootdisp=17.497,
refid=10.71.0.253,
reftime=ecd431f6.969e8de5  Fri, Nov 28 2025 11:36:38.588,
clock=ecd434e9.33b2aa83  Fri, Nov 28 2025 11:49:13.201, peer=43923,
tc=8, mintc=3, offset=+0.233599, frequency=+32.658, sys_jitter=0.000000,
clk_jitter=0.407, clk_wander=0.066

root@R1> show ntp associations
     remote               refid           auth  st  t  when  poll reach  delay     offset   jitter rootdelay rootdisp
=====================================================================================================================
*srv1.jncie.lab         10.10.10.10      SKEY   3  u   225   256  377    0.762    +0.234    0.062   12.680    2.289

root@R1> show system uptime
Current time: 2025-11-28 11:49:20 -03
Time Source:  NTP CLOCK
System booted: 2025-11-24 11:21:17 -03 (4d 00:28 ago)
Protocols started: 2025-11-24 11:25:44 -03 (4d 00:23 ago)
Last configured: 2025-11-28 11:20:38 -03 (00:28:42 ago) by root
11:49AM  up 4 days, 28 mins, 1 users, load averages: 1.03, 1.03, 1.15
```
Everything’s synced, and yes, that is my actual timezone, okay?

And what about backups?
How do we recover our configuration if something goes wrong?

The easiest way is to configure our router to automatically send a copy of the config every time we do a commit. On my SRV-1, I created a user just for FTP access, so the router can upload the files without any issues.

Here’s the configuration:
```
set system archival configuration transfer-on-commit
set system archival configuration archive-sites "ftp://lab@10.71.0.253" password lab123
```
Now let’s check if the backup actually landed on the server:
```
root@kvm:/srv/ftp/backups# ls -lh
total 1.2K
-rw-r--r-- 1 lab  lab  1.2K Nov 24 16:15 R1_20251124_191441_juniper.conf.gz
```
And there it is, nice and clean.

Next step: let’s create some local users to log into the routers.
We’ll simulate a few different profiles so we can play with privilege levels and see how flexible Junos can be when it comes to user hierarchy.

Here’s how our users and their permissions will look:
| Username | Password | Privileges |
| -------- | -------- | ---------- |
| lab | lab123 | super-user |
| any | - | view + view-configuration (authentication via RADIUS only) |
| support | suport123 | clear, network, reset, trace, and view (operator class)| 
| noc | noc123 | basically everything except clear, configure, edit, or starting the shell |

The configuration for the lab user is shown below:
```
set system login user lab class super-user
set system login user lab authentication plain-text-password lab123
```
Let`s test it:
```
lab@R1> show cli authorization
Current user: 'lab         ' class 'super-user'
```
Now, we need to create a class for any user that logs in via RADIUS. When a user authenticates through RADIUS, they are mapped to the remote user, and this user will belong to the view-only class.
```
set system login view-only limited permissions view
set system login view-only limited permissions view-configuration
set system login user remote class view-only
```
But, we don't have any RADIUS server configuration yet. 
Let's do it!
First of all, we have to change the authentication-order to try authenticating the user on the RADIUS server first. If that fails, the router will then try to authenticate locally.
```
set system authentication-order [ radius local ]
```

For the RADIUS server configuration, we need to specify the shared secret, and we can customize the number of retries and timeouts.
If we hit two timeouts during the requests, the authentication will fall back to local.
And if the RADIUS server responds but the authentication fails, the router will retry once. If it still fails, it will also fall back to local authentication.
```
set system radius-server 10.71.0.253 secret L4B
set system radius-server 10.71.0.253 timeout 2
set system radius-server 10.71.0.253 retry 1
```

Now, let`s test the authentication via RADIUS:
```
--- JUNOS 24.4R1.9 Kernel 64-bit  JNPR-15.0-20241104.1ed86e6_buil
etorri@R1> show cli authorization
Current user: 'remote' login: 'etorri' class 'view-only'
Permissions:
    view        -- Can view current values and statistics
    view-configuration-- Can view all configuration (not including secrets)
```
As we can see above, my user has been assigned the view-only class because I logged in via RADIUS.

Since we’re on this topic, let’s turn my user into a super-user.
On the RADIUS server, we’ll add the Juniper-Local-User-Name attribute. This way, when the user etorri authenticates via RADIUS, Junos will look for a matching local user to assign its privileges:
```
etorri Cleartext-Password := "lab123"
    Juniper-Local-User-Name := "SU"
```
And on Junos, let’s create the local user SU. 
```
set system login user SU class super-user
```
And now, let's test it:
```
--- JUNOS 24.4R1.9 Kernel 64-bit  JNPR-15.0-20241104.1ed86e6_buil
etorri@R1> show cli authorization
Current user: 'SU' login: 'etorri' class 'super-user'
```
Sucess!!!

Now, let's continue creating the other users. 
The support user will have the operator class.
```
set system login user support class operator
set system login user support authentication plain-text-password support123
```
Let’s check this user as well:
```
--- JUNOS 24.4R1.9 Kernel 64-bit  JNPR-15.0-20241104.1ed86e6_buil
support@R1> show cli authorization
Current user: 'support     ' class 'operator'
```
And finally, the noc user will have all permissions except clear, configure, edit, and start shell.
This way we can explore Junos’ flexibility to restrict specific commands. See below:
```
set system login class noc permissions all
set system login class noc deny-commands "(clear)|(configure)|(edit)|(start shell)"
set user noc class noc authentication plain-text-password noc123
```
We created a regular expression where the pipe | represents OR, so Junos will match any command inside the expression.
This means the user can run all commands except the ones explicitly denied.

Let’s check the authorization:
```
--- JUNOS 24.4R1.9 Kernel 64-bit  JNPR-15.0-20241104.1ed86e6_buil
noc@R1> show cli authorization
Current user: 'noc         ' class 'noc'
...
Individual command authorization:
    Allow regular expression: none
    Deny regular expression: (clear)|(configure)|(edit)|(start shell)
    Allow configuration regular expression: none
    Deny configuration regular expression: none
```
By checking the authorization output, we can clearly see which commands are not allowed for this user.

Finally, we will configure the log messages and define what will be sent to the syslog server.
Let’s follow the parameters below:
| Receiver | Message type |
| ------- | ------------ |
| messages archive | All info-level messages |
| user-commands archive | All commands/interactive-commands |
| ops user | all notice-level messages |
All archive files should keep 3 versions, with 100 kb each.

Let's do it:
```
set system syslog archive files 3 size 100k
set system syslog file messages any info
set system syslog file user-commands interactive-commands any
set system syslog user ops any warning
```
This way, all the archive will keep 3 versions, with 100kb each. The messages file will store only info-level messages, the user-commands file will store every interactive command executed on Junos. And the ops user will see warning-level messages directly in the terminal after logging in. 

For the syslog configuration, we need to send all notice-level messages, all interactive commands, and the change logs as well. 
Our syslog server will be the SRV-1, the multirole server, the brave, the best, glorious!!!
```
set system syslog host 10.10.1.253 any notice
set system syslog host 10.10.1.253 change-log any
set system syslog host 10.10.1.253 interactive-commands any
```
This way, our server will receive these logs from all routers in our network.
Now, let’s check R1's logs:
```
root@kvm:/# tail -f /var/log/lab-jncie/R1.log
Nov 28 14:12:38 R1 mgd[88299]: UI_AUTH_EVENT: Authenticated user 'support' assigned to class 'j-operator'
Nov 28 14:12:38 R1 mgd[88299]: UI_LOGIN_EVENT: User 'support' login, class 'j-operator' [88299], ssh-connection '10.71.0.254 34882 10.71.0.1 22', client-mode 'cli'
```

This is the first part of our JNCIE-SP journey.
Yeah, I know... this section was pretty basic, so you might think it’s a bit boring.
But hold on… the good stuff is coming! 


