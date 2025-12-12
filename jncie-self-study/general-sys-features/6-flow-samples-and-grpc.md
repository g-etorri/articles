# Flow Sampling and gRPC for Telemetry Transmission

Hello guys,

Today, we'll configure flow sampling on R5 to capture some packets from DC3 (this is a new addition to our topology, but I won't cover it right now).

I'll use IPFIX to send the flow samples to SRV1, which requires configuring a new interface between them.

So, first, we need to establish connectivity with SRV1. I've set aside the network 10.10.11.0/24 for this purpose. I also have plans to deploy a DNS server for our customers on this network later on. However, for this lab, I'll use it strictly to connect R5 to SRV1 and export the flow samples.
```
set interfaces ge-0/0/4 description to-SRV1/eth1
set interfaces ge-0/0/4 unit 0 family inet address 10.10.11.1/24

root@R5> ping 10.10.11.2
PING 10.10.11.2 (10.10.11.2): 56 data bytes
64 bytes from 10.10.11.2: icmp_seq=0 ttl=64 time=159.553 ms
64 bytes from 10.10.11.2: icmp_seq=1 ttl=64 time=1.615 ms
64 bytes from 10.10.11.2: icmp_seq=2 ttl=64 time=2.168 ms
64 bytes from 10.10.11.2: icmp_seq=3 ttl=64 time=1.843 ms
^C
--- 10.10.11.2 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.615/41.295/159.553/68.277 ms
```
Alright, we’ve got connectivity!

Now, it's time to build the templates for the inet and inet6 families. Here is the logic we're going to use:

Active Timeout (30s): We’ll set this to 30 seconds. Basically, if a flow is long-lived (like a continuous stream), the router will export a record every 30 seconds to update the collector.

Inactive Timeout (180s): If the traffic stops and we hear silence for 180 seconds, the flow is considered dead (inactive) and flushed from the cache.

We’re going to apply the same configuration for both families. Let’s do it.
```
set services flow-monitoring version-ipfix template SRV1-IPv6-IPFIX flow-active-timeout 30
set services flow-monitoring version-ipfix template SRV1-IPv6-IPFIX flow-inactive-timeout 180
set services flow-monitoring version-ipfix template SRV1-IPv6-IPFIX ipv6-template
set services flow-monitoring version-ipfix template SRV1-IPv4-IPFIX flow-active-timeout 30
set services flow-monitoring version-ipfix template SRV1-IPv4-IPFIX flow-inactive-timeout 180
set services flow-monitoring version-ipfix template SRV1-IPv4-IPFIX ipv4-template
```
Alright, templates are defined! Now, let’s jump into forwarding-options to configure the sampling instance.

We’re setting the sampling rate to 1000 (meaning 1 out of every 1000 packets gets picked). To keep performance tight, we’re going to offload this to the PFE using inline-jflow. This ensures the sampling happens directly in hardware—keeping our RE CPU happy!

One specific detail for this project is how we handle destination ports:

inet samples will be sent to UDP 44321

inet6 samples go to UDP 46321

With that in mind, let's look at the config. You’ll notice we need to specify the source address and map the templates here, but knowing you guys rely on Junos muscle memory, this should be second nature.
```
set forwarding-options sampling instance to-SRV1 input rate 1000
set forwarding-options sampling instance to-SRV1 family inet output flow-server 10.10.11.2 port 44321
set forwarding-options sampling instance to-SRV1 family inet output flow-server 10.10.11.2 source-address 10.0.0.5
set forwarding-options sampling instance to-SRV1 family inet output flow-server 10.10.11.2 version-ipfix template SRV1-IPv4-IPFIX
set forwarding-options sampling instance to-SRV1 family inet output inline-jflow source-address 10.0.0.5
set forwarding-options sampling instance to-SRV1 family inet6 output flow-server 10.10.11.2 port 46321
set forwarding-options sampling instance to-SRV1 family inet6 output flow-server 10.10.11.2 source-address 10.0.0.5
set forwarding-options sampling instance to-SRV1 family inet6 output flow-server 10.10.11.2 version-ipfix template SRV1-IPv6-IPFIX
set forwarding-options sampling instance to-SRV1 family inet6 output inline-jflow source-address 10.0.0.5
```
Everything looks solid so far. But for this to actually work, we need to tie the sampling instance to the FPC where our interface lives.

Once that's done, we just need to specify on which interface we want to start collecting those flows.
```
set interfaces ge-0/0/6 description to-DC3/ge-0/0/1
set interfaces ge-0/0/6 unit 0 family inet sampling input
set interfaces ge-0/0/6 unit 0 family inet sampling output
set interfaces ge-0/0/6 unit 0 family inet address 172.30.130.5/30
set interfaces ge-0/0/6 unit 0 family inet6 sampling input
set interfaces ge-0/0/6 unit 0 family inet6 sampling output
set chassis fpc 0 sampling-instance to-SRV1
```
With this configuration, we can now see the flows arriving at SRV1!

Let's send some pings to test it out.
```
root@kvm:/usr/sbin# tcpdump -i eth1 'port 46321 or port 44321' -vvv -X
tcpdump: listening on eth1, link-type EN10MB (Ethernet), capture size 262144 bytes
09:39:21.558154 IP (tos 0x0, ttl 250, id 57810, offset 0, flags [none], proto UDP (17), length 138)
    10.0.0.5.50101 > 10.10.11.2.44321: [no cksum] UDP, length 110
        0x0000:  4500 008a e1d2 0000 fa11 bf7f 0a00 0005  E...............
        0x0010:  0a0a 0b02 c3b5 ad21 0076 0000 000a 006e  .......!.v.....n
        0x0020:  6939 69f8 0000 0009 0008 0000 0100 005e  i9i............^
        0x0030:  ac1e 8205 ac1e 8206 0001 0000 0000 0800  ................
        0x0040:  0000 0233 0000 201e 0000 fdfc 0000 fdfc  ...3............
        0x0050:  0000 0000 0000 0002 3340 4002 0400 0000  ........3@@.....
        0x0060:  00ff 0000 0000 0000 0000 0000 0000 0000  ................
        0x0070:  0054 0000 0000 0000 0001 0000 019b 0845  .T.............E
        0x0080:  7c00 0000 019b 0845 7c00                 |......E|.
09:39:25.640754 IP (tos 0x0, ttl 250, id 6000, offset 0, flags [none], proto UDP (17), length 138)
    10.0.0.5.50101 > 10.10.11.2.44321: [no cksum] UDP, length 110
        0x0000:  4500 008a 1770 0000 fa11 89e2 0a00 0005  E....p..........
        0x0010:  0a0a 0b02 c3b5 ad21 0076 0000 000a 006e  .......!.v.....n
        0x0020:  6939 69fc 0000 000a 0008 0000 0100 005e  i9i............^
        0x0030:  ac1e 8205 ac1e 8206 0001 0000 0000 0000  ................
        0x0040:  0000 0000 0000 201e 0000 fdfc 0000 fdfc  ................
        0x0050:  0000 0000 0000 0002 3340 4002 0400 0000  ........3@@.....
        0x0060:  00ff 0000 0000 0000 0000 0000 0000 0000  ................
        0x0070:  0054 0000 0000 0000 0001 0000 019b 0845  .T.............E
        0x0080:  8d00 0000 019b 0845 8d00                 .......E..
09:39:35.627258 IP (tos 0x0, ttl 250, id 13953, offset 0, flags [none], proto UDP (17), length 189)
    10.0.0.5.50102 > 10.10.11.2.46321: [no cksum] UDP, length 161
        0x0000:  4500 00bd 3681 0000 fa11 6a9e 0a00 0005  E...6.....j.....
        0x0010:  0a0a 0b02 c3b6 b4f1 00a9 0000 000a 00a1  ................
        0x0020:  6939 6a06 0000 00e3 0009 0000 0101 0091  i9j.............
        0x0030:  fe80 0000 0000 0000 52c8 d8ff fe00 0f08  ........R.......
        0x0040:  fe80 0000 0000 0000 0205 86ff fe71 8d01  .............q..
        0x0050:  003a 0000 0000 8000 0000 0233 0000 8000  .:.........3....
        0x0060:  0000 fdfc ffff ffff 0000 0000 0000 0000  ................
        0x0070:  0000 0000 0000 0000 0000 0000 0000 0000  ................
        0x0080:  0000 0000 0000 0000 0000 0000 0040 4002  .............@@.
        0x0090:  ff00 0000 0000 0000 0000 0000 0000 0000  ................
        0x00a0:  0000 0001 1800 0000 0000 0000 0500 0001  ................
        0x00b0:  9b08 45b3 0000 0001 9b08 45be 00         ..E.......E..
09:39:37.668597 IP (tos 0x0, ttl 250, id 65328, offset 0, flags [none], proto UDP (17), length 189)
    10.0.0.5.50102 > 10.10.11.2.46321: [no cksum] UDP, length 161
        0x0000:  4500 00bd ff30 0000 fa11 a1ee 0a00 0005  E....0..........
        0x0010:  0a0a 0b02 c3b6 b4f1 00a9 0000 000a 00a1  ................
        0x0020:  6939 6a08 0000 00e4 0009 0000 0101 0091  i9j.............
        0x0030:  fe80 0000 0000 0000 52c8 d8ff fe00 0f08  ........R.......
        0x0040:  fe80 0000 0000 0000 0205 86ff fe71 8d01  .............q..
        0x0050:  003a 0000 0000 8100 0000 0000 0000 8000  .:..............
        0x0060:  0000 fdfc ffff ffff 0000 0000 0000 0000  ................
        0x0070:  0000 0000 0000 0000 0000 0000 0000 0000  ................
        0x0080:  0000 0000 0000 0000 0000 0000 0040 4002  .............@@.
        0x0090:  ff00 0000 0000 0000 0000 0000 0000 0000  ................
        0x00a0:  0000 0000 7000 0000 0000 0000 0200 0001  ....p...........
        0x00b0:  9b08 45ba 0000 0001 9b08 45bc 00         ..E.......E..
^C
4 packets captured
4 packets received by filter
0 packets dropped by kernel
```
And... Voila! We are receiving flow samples for both inet and inet6.

For this lab, I don't want to get bogged down setting up a fancy IPFIX collector application. A simple tcpdump is enough proof for me right now.

Now, shifting gears to Telemetry.

I really want to deep dive into telemetry for automation, it's absolutely essential these days.

I’ve configured Telegraf on SRV1 to connect to R5 via gRPC and stream the data. Basically, Telegraf requests the info, and the router pushes it back.

Now, I won’t write a full tutorial on configuring Telegraf (I'm not a Telegraf guru yet!), but I will share my input plugin configuration below. I'm using the gNMI plugin.

First of all, you need to know what you actually want to [collect](https://apps.juniper.net/telemetry-explorer/).

For demonstration purposes, I'll just grab the interface descriptions, but feel free to get creative. Let's go:
```
[[inputs.gnmi]]
  addresses = ["10.0.0.5:43123"]
  username = "telegraf"
  password = "Telegraf123"

  encoding = "proto"
  redial = "10s"


  [[inputs.gnmi.subscription]]
    name = "interfaces_stats"
    origin = "openconfig"
    path = "/interfaces/interface/state/description"
    subscription_mode = "sample"
    sample_interval = "180s"
```
My Telegraf instance will connect to R5 on port 43123 using the credentials telegraf / Telegraf123. Obviously, we need to create this user on the router first.

The input plugin is the piece of the puzzle that requests data from our router. It uses a Sensor Path to locate the data, think of this path as the modern equivalent of an SNMP OID. I'm setting it to pull data every 180 seconds.

On the router side, we need to configure the user and the gRPC parameters.

Note: Since this is just a lab environment, I’m skipping SSL to keep things simple. In the real world, please encrypt your telemetry traffic! But for now, a basic clear-text configuration will do.
```
set system login user telegraf class super-user
set system login user telegraf authentication plain-text-password Telegraf123
set system services extension-service request-response grpc clear-text port 43123
set system services extension-service request-response grpc max-connections 15
set system services extension-service request-response grpc skip-authentication
set system services extension-service notification allow-clients address 10.10.11.2/32
```

Now, let's fire up Telegraf and start pulling this data:
```
root@kvm:/etc/telegraf# telegraf --config /etc/telegraf/telegraf.conf --debug
2025-12-11T20:13:13Z W! Strict environment variable handling will be the new default starting with v1.38.0! If your configu                                                              ration works with strict handling or you don't use environment variables it is safe to ignore this warning. Otherwise pleas                                                              e explicitly add the --non-strict-env-handling flag!
2025-12-11T20:13:13Z I! Loading config: /etc/telegraf/telegraf.conf
...
2025-12-11T20:13:13Z D! [inputs.gnmi] Connection to gNMI device 10.0.0.5:43123 established
2025-12-11T20:13:23Z D! [outputs.file] Wrote batch of 52 metrics in 933.596µs
2025-12-11T20:13:23Z D! [outputs.file] Buffer fullness: 0 / 10000 metrics
^C2025-12-11T20:13:31Z D! [agent] Stopping service inputs
2025-12-11T20:13:31Z D! [inputs.gnmi] Connection to gNMI device 10.0.0.5:43123 closed
```
No errors popped up, so it looks like smooth sailing! I'm assuming the data made it through.

Let's take a look at the output file to confirm:
```
root@kvm:/etc/telegraf# cat /tmp/telemetry_junos.out | grep to-
{"fields":{"description":"to-R6/ge-0/0/0"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/0","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484099}
{"fields":{"description":"to-R6/ge-0/0/1"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/1","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484099}
{"fields":{"description":"to-R8/ge-0/0/2"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/2","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484099}
{"fields":{"description":"to-R4/ge-0/0/3"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/3","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484099}
{"fields":{"description":"to-SRV1/eth1"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/4","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484099}
{"fields":{"description":"to-DC2/ge-0/0/1"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/5","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484099}
{"fields":{"description":"to-DC3/ge-0/0/1"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/6","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484099}
{"fields":{"description":"to-R6/ae0"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ae0","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484099}
{"fields":{"description":"to-R6/ge-0/0/0"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/0","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484109}
{"fields":{"description":"to-R6/ge-0/0/1"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/1","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484109}
{"fields":{"description":"to-R8/ge-0/0/2"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/2","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484109}
{"fields":{"description":"to-R4/ge-0/0/3"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/3","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484109}
{"fields":{"description":"to-SRV1/eth1"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/4","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484109}
{"fields":{"description":"to-DC2/ge-0/0/1"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/5","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484109}
{"fields":{"description":"to-DC3/ge-0/0/1"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ge-0/0/6","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484109}
{"fields":{"description":"to-R6/ae0"},"name":"interfaces_stats","tags":{"host":"kvm","name":"ae0","path":"/interfaces/interface","source":"10.0.0.5"},"timestamp":1765484109}
root@kvm:/etc/telegraf# cat /tmp/telemetry_junos.out | grep to- | jq .
{
  "fields": {
    "description": "to-R6/ge-0/0/0"
  },
  "name": "interfaces_stats",
  "tags": {
    "host": "kvm",
    "name": "ge-0/0/0",
    "path": "/interfaces/interface",
    "source": "10.0.0.5"
  },
  "timestamp": 1765484149
}
{
  "fields": {
    "description": "to-R6/ge-0/0/1"
  },
  "name": "interfaces_stats",
  "tags": {
    "host": "kvm",
    "name": "ge-0/0/1",
    "path": "/interfaces/interface",
    "source": "10.0.0.5"
  },
  "timestamp": 1765484149
}
{
  "fields": {
    "description": "to-R8/ge-0/0/2"
  },
  "name": "interfaces_stats",
  "tags": {
    "host": "kvm",
    "name": "ge-0/0/2",
    "path": "/interfaces/interface",
    "source": "10.0.0.5"
  },
  "timestamp": 1765484149
}
{
  "fields": {
    "description": "to-R4/ge-0/0/3"
  },
  "name": "interfaces_stats",
  "tags": {
    "host": "kvm",
    "name": "ge-0/0/3",
    "path": "/interfaces/interface",
    "source": "10.0.0.5"
  },
  "timestamp": 1765484149
}
{
  "fields": {
    "description": "to-SRV1/eth1"
  },
  "name": "interfaces_stats",
  "tags": {
    "host": "kvm",
    "name": "ge-0/0/4",
    "path": "/interfaces/interface",
    "source": "10.0.0.5"
  },
  "timestamp": 1765484149
}
{
  "fields": {
    "description": "to-DC2/ge-0/0/1"
  },
  "name": "interfaces_stats",
  "tags": {
    "host": "kvm",
    "name": "ge-0/0/5",
    "path": "/interfaces/interface",
    "source": "10.0.0.5"
  },
  "timestamp": 1765484149
}
{
  "fields": {
    "description": "to-DC3/ge-0/0/1"
  },
  "name": "interfaces_stats",
  "tags": {
    "host": "kvm",
    "name": "ge-0/0/6",
    "path": "/interfaces/interface",
    "source": "10.0.0.5"
  },
  "timestamp": 1765484149
}
```
Wooow!!! Data is flowing in perfectly! Now we are ready to collect all sorts of metrics to automate and monitor our routers.

Honestly, the sky's the limit here. You can use your creativity to do wonderful things with telemetry. I’ve actually tried [OpenJTS](https://github.com/door7302/openjts), it's an amazing tool and even comes with some pre-built templates to grab data from your gear.

And with that, we’ve wrapped up the 'System Features' section of our JNCIE journey.

Next stop: IGP configuration!!!

See you in the next one, bye!
