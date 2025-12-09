# Junos Scripting

Hello everyone, today we’re going to take a look at some scripts we can run on Junos.
We’ll go through SLAX scripts, which, honestly, I can’t stand… arrghhh, and then we’ll look at some Python scripts, which are much better. So much better.

First things first: we need to explain the types of scripts we can run on Junos.
There are three categories:
Operational scripts, also called “op” scripts.
Commit scripts, which run every time we commit a configuration.
Event scripts, which are triggered by specific events inside Junos.

These scripts give us flexible ways to customize outputs, automate tasks, and generally make our troubleshooting and day-to-day operations a lot easier.

Now, listen… I don’t want to learn how to write SLAX scripts.
I really don’t.
So, with that mindset, I went searching for some ready-made scripts. Here’s the link:
https://github.com/Juniper/junoscriptorium/tree/master/library/juniper

I’ll grab a few SLAX scripts from there and test them out. Yes, yes, I know... you were probably expecting more from this topic. But I have to confess something to you… I hate SLAX. Okay?

Before testing anything, we need to transfer the script files into their specific directories on the router:
Op scripts > /var/db/scripts/op
Commit scripts > /var/db/scripts/commit
Event scripts > /var/db/scripts/event

Alright, let’s test an op script.
I’ll pick the [show-interfaces.slax](https://github.com/Juniper/junoscriptorium/blob/master/library/juniper/op/display/show-interfaces/show-interfaces.slax) script:
```
root@R1> file copy ftp://lab:lab123@10.71.0.253/scripts/show-interfaces.slax /var/db/scripts/op/
/var/tmp//...transferring.file.........42TFL5/100% of 2775  B 1655 kBps 00m00s

root@R1> configure
Entering configuration mode

[edit]
root@R1#set system scripts op file show-interfaces.slax
root@R1> op show-interfaces | except "jsrv|em1|1638|3276"
Interface               Admin Link Proto    Local                 Remote
ge-0/0/0.0              to-R2/ge-0/0/0
                                   aenet    --> ae0.0
                                   inet6
ge-0/0/1.0              to-R2/ge-0/0/1
                                   aenet    --> ae0.0
ge-0/0/2.0              to-R4/ge-0/0/2
                                   inet     10.200.0.2          --> 10.200.0.2/31
                                   iso
                                   multiservice
ge-0/0/3.0              to-R8/ge-0/0/3
                                   inet     10.200.0.4          --> 10.200.0.4/31
                                   iso
                                   inet6    fe80::526b:25ff:fe00:605--> fe80::/64
                                   multiservice
ae0.0                   to-R2/ae0
                                   inet     10.200.0.0          --> 10.200.0.0/31
                                   iso
                                   inet6    fe80::2e6b:f5ff:fe48:61c0--> fe80::/64
                                   multiservice
                                            128.0.0.1           --> 128/2
                                   inet6    128.0.0.4           --> 128/2
                                            fe80::523e:99ff:fe00:501--> fe80::/64
                                   tnp      fec0::a:0:0:4       --> fec0::/64
                                            0x4
fxp0.0                  MGMT-OOB
                                   inet     10.71.0.1           --> 10.71.0/24
lo0.0                              inet     10.0.0.1
                                   iso      49.0001.0100.0000.0001
                                   inet6    fd10:faca:f0fa::1
                                            fe80::523e:990f:fc00:500
```
This is basically a “show interface terse” on steroids!
I still need to filter out some junk in the output, but overall it works nicely.

With this script, we can see the interface description, the local and remote IP addresses, and in the case of ae interfaces, we can even see the bundle members. For operational routines, this is gold. It gives you a compact but meaningful summary of the interface state, exactly what you want when you’re troubleshooting under pressure.
Alright, now that you’ve seen the value of op scripts, let’s move on to commit scripts.

Here’s the way to think about commit scripts: they act like automated rule enforcers.
If you define an internal policy such as “In my network, we do NOT allow prefixes bigger than /24”, a commit script can enforce that rule by inspecting the candidate configuration. If you try to configure a /23, the script checks it and throws a warning on your screen before allowing or rejecting the commit.

To demonstrate this, I’m going to use the script [interface-mask-check.slax](https://github.com/Juniper/junoscriptorium/blob/master/library/juniper/commit/interfaces/interface-mask-check/interface-mask-check.slax):
This script does exactly what I described, it inspects interface configurations and warns you when an interface has a mask that violates your internal mask policy.
Let’s go ahead and test it.
```
root@R1> file copy ftp://lab:lab123@10.71.0.253/scripts/interface-mask-check.slax /var/db/scripts/commit/
/var/tmp//...transferring.file.........n8y9rL/100% of 1021  B  754 kBps 00m00s

root@R1> configure
Entering configuration mode

[edit]
root@R1# set system scripts commit file interface-mask-check.slax

[edit]
root@R1# show | compare
[edit interfaces]
+   ge-0/0/5 {
+       unit 0 {
+           family inet {
+               address 192.168.0.1/23;
+           }
+       }
+   }

[edit]
root@R1# commit check
warning: The address of 192.168.0.1 has a mask of /23
on interface ge-0/0/5 unit 0
configuration check succeeds
```
Yeah, this one works perfectly! I know it’s probably one of the simplest scripts in the whole repo, but that’s exactly the point, it’s just to get the feel of how commit scripts behave.
There are tons of other scripts available, and you can explore all of them in the link I shared earlier.

Now let’s move to what I think is the coolest category of all:
✨ Event scripts! ✨

These are awesome because they let you hook directly into Junos events and take action whenever something specific happens on the router. You can essentially turn logs into triggers. For example, when an interface goes down, you can generate a custom syslog message… or even fire off a brand-new SNMP trap that you created.

Let’s check out a practical example.

We’ll use the script: [syslog-int-desc-on-link-change.slax](https://github.com/Juniper/junoscriptorium/blob/master/library/juniper/event/interfaces/syslog-int-desc-on-link-change/syslog-int-desc-on-link-change.slax)

Event scripts behave a bit differently.
In this case, the script creates a new syslog message every time a link changes state. And this message doesn’t just say “interface down”, it includes the interface and its description. That’s incredibly useful for operations, because you immediately see what service or customer is affected.

Once we have this custom syslog event, we can go even further:
we can build an SNMP trap triggered by this exact message.
This means you can notify your Zabbix with precise, enriched information, far better than a generic interface down trap.

So, let’s continue with the steps.
First, we copy the script, and now we need to activate it so Junos can actually use it.
```
root@R1> file copy ftp://lab:lab123@10.71.0.253/scripts/syslog-int-desc-on-link-change.slax /var/db/scripts/event/
/var/tmp//...transferring.file.........OGRR4S/100% of 5207  B 4421 kBps 00m00s

root@R1> configure
Entering configuration mode

[edit]
root@R1#set event-options event-script file syslog-int-desc-on-link-change.slax
```
With the script in place, we can finally put it to work. Now it’s time to create the custom syslog message that the router will generate whenever an interface changes state.
```
set event-options policy syslog_if_description events snmp_trap_link_up
set event-options policy syslog_if_description events snmp_trap_link_down
set event-options policy syslog_if_description then event-script syslog-int-desc-on-link-change.slax
```
Basically, when a link state changes, the event script fires instantly and injects a brand-new syslog entry into the router’s log. This message is much richer than the default one because it includes the interface description, which is exactly what we want for cleaner troubleshooting and better visibility.
```
[edit]
root@R1# set interfaces ae0 disable

[edit]
root@R1# commit and-quit
kenv: unable to get vmtype
commit complete
Exiting configuration mode

root@R1> configure
Entering configuration mode

[edit]
root@R1# rollback 1
load complete

syntax error, expecting <command>.
root@R1# show | compare
[edit interfaces ae0]
-   disable;

[edit]
root@R1# commit and-quit
kenv: unable to get vmtype
commit complete
Exiting configuration mode

root@R1> show log messages | last 50 | match "NEW_SNMP"
Dec  9 08:34:19  R1 cscript[14180]: NEW_SNMP_TRAP_LINK_DOWN, R1, ae0, down, down, to-R2/ae0
Dec  9 08:34:19  R1 cscript[14181]: NEW_SNMP_TRAP_LINK_DOWN, R1, ge-0/0/0, down, down, to-R2/ge-0/0/0
Dec  9 08:34:19  R1 cscript[14182]: NEW_SNMP_TRAP_LINK_DOWN, R1, ge-0/0/1, down, down, to-R2/ge-0/0/1
Dec  9 08:34:33  R1 mgd[33654]: UI_CMDLINE_READ_LINE: User 'root', command 'show log messages | last 50 | match NEW_SNMP '
Dec  9 08:39:37  R1 cscript[14330]: NEW_SNMP_TRAP_LINK_UP, R1, , , ,
Dec  9 08:39:37  R1 cscript[14332]: NEW_SNMP_TRAP_LINK_UP, R1, , , ,
Dec  9 08:39:37  R1 cscript[14329]: NEW_SNMP_TRAP_LINK_UP, R1, ge-0/0/0, up, up, to-R2/ge-0/0/0
Dec  9 08:39:37  R1 cscript[14331]: NEW_SNMP_TRAP_LINK_UP, R1, ge-0/0/1, up, up, to-R2/ge-0/0/1
Dec  9 08:39:38  R1 mgd[33654]: UI_CMDLINE_READ_LINE: User 'root', command 'show log messages | last 50 | match NEW_SNMP '
Dec  9 08:39:38  R1 cscript[14342]: NEW_SNMP_TRAP_LINK_UP, R1, , , ,
```
This is working perfectly, and honestly, I think this is the most powerful type of script Junos gives us. Event scripts are just insane, you can bend the router to your will. And yes, we can work with Python too, and we’ll take a look at that right after this.

Now that our custom syslog message is being generated every time the link state changes, we can take the next step:
create a brand-new SNMP TRAP based on this custom syslog entry.
This is where things get fun.

Here we go!
```
set event-options policy snmptrap_if_description events SYSTEM
set event-options policy snmptrap_if_description attributes-match SYSTEM.message matches NEM_SNMP_TRAP_LINK
set event-options policy snmptrap_if_description then raise-trap
```
With this in place, every time a link changes state, we’ll get a fresh syslog message and a brand-new SNMP TRAP fired off to our SRV1. Clean and simple.

Alright, now let’s jump into the Python side of things.
I’m going to use a script that disables any interface we choose.
If you want to check out the code, take a look at [interface_disable.py](https://sipart.github.io/Junos_Python_Notes/):

This script basically asks which interface you want to shut down, once you confirm, boom, it disables it.
```
root@R1> file copy ftp://lab:lab123@10.71.0.253/scripts/interface_disable.py /var/db/scripts/op
/var/tmp//...transferring.file.........1YoZse/100% of 1181  B  825 kBps 00m00s

[edit system scripts]
root@R1# set language python3

[edit system scripts]
root@R1# set op file interface_disable.py

[edit system scripts]
root@R1# commit and-quit
kenv: unable to get vmtype
commit complete
Exiting configuration mode

root@R1> op interface_disable.py
        This script disables the specified interface.
        The script modifies the candidate configuration to disable
        the interface and commits the configuration to activate it.
Enter interface to disable: ge-0/0/0
Loading and committing configuration changes

root@R1> show configuration interfaces ge-0/0/0
description to-R2/ge-0/0/0;
disable;
gigether-options {
    802.3ad ae0;
}
```
And… voilà. Yeah, this one’s dangerous, no doubt about it. But if you use it wisely, it becomes a really powerful tool for day-to-day operations.

So that’s everything I wanted to cover about scripting on Junos for now.
Next up, we’ll dive into flow sampling and gRPC for Telemetry!!!
And finally, we’ll get our IGP up and running to deliver some services to our pseudo-customers!

Bye bye.


