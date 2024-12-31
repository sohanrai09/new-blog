---
title: "Model Driven Telemetry in Cisco IOS-XR"
date: 2024-12-31T18:28:20+05:30
categories:
- telemetry
- cisco
#- subcategory
tags:
- telemetry
- cisco
# - tag2
keywords:
- tech
thumbnailImagePosition: left
thumbnailImage: "https://images.pexels.com/photos/187041/pexels-photo-187041.jpeg?auto=compress&cs=tinysrgb&w=1260&h=750&dpr=2"
coverImage: "https://media.istockphoto.com/id/1488294044/photo/businessman-works-on-laptop-showing-business-analytics-dashboard-with-charts-metrics-and-kpi.jpg?s=1024x1024&w=is&k=20&c=VpSNiVam6Fw3egrJYnP28mEEAXyCjFRjqV_k4PK5S04=
---

I know I'm very late to the party and probably everbody has already moved to Telemetry, but during the Annual shutdown of Dec '24, I wanted to set this up in my personal lab and play around. I have been testing Model Driven Telemetry(MDT) on Cisco's IOS-XR for a while, so I wanted to share some of my learnings to help someone along their way for setting up telemetry. I will be primarily focusing on Cisco IOS-XR's implementation of MDT, more precisely GRPC DIAL-OUT for now.

There are already tons of content on setting up the TIG(Telegraf, InfluxDB, Grafana) stack, I just wanted to mention here the Telegraf configuration for a quick reference.

```
# telegraf.conf
# Global Agent Configuration
[agent]
hostname = "telemetry-container"
flush_interval = "15s"
interval = "15s"

# gRPC Dial-Out Telemetry Listener
[[inputs.cisco_telemetry_mdt]]
transport = "grpc"
service_address = ":57000"

# Output Plugin InfluxDB
[[outputs.influxdb_v2]]
bucket = "mdtlab"
urls = ["http://127.0.0.1:8086"]
token = "enter the token here"
organization = "lab"
```


### Configuration on IOS-XR
As mentioned earlier, I wil be using `grpc dial-out` option, and the required configuration on Cisco IOS-XR for this:
```
!
grpc
 no-tls
!
telemetry model-driven
 destination-group DGroup2
  address-family ipv4 192.168.100.1 port 57000
   encoding self-describing-gpb
   protocol grpc no-tls
  !
 !
 sensor-group ISIS
  sensor-path Cisco-IOS-XR-clns-isis-oper:isis/instances/instance/summary
  sensor-path Cisco-IOS-XR-ip-rib-ipv4-oper:rib/vrfs/vrf/afs/af/safs/saf/ip-rib-route-table-names/ip-rib-route-table-name/protocol/isis/as/information
 !
 sensor-group MPLSTE
  sensor-path Cisco-IOS-XR-mpls-te-oper:mpls-te/tunnels/summary
 !
 sensor-group SGroup2
  sensor-path Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring/cpu-utilization
  sensor-path Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary
 !
 sensor-group ipv4_bgp
  sensor-path Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/vrfs/vrf/process-info/global
  sensor-path Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/process-info/global
  sensor-path Cisco-IOS-XR-ip-rib-ipv4-oper:rib/vrfs/vrf/afs/af/safs/saf/ip-rib-route-table-names/ip-rib-route-table-name/protocol/bgp/as
 !
 sensor-group rib-ipv4
  sensor-path Cisco-IOS-XR-ip-rib-ipv4-oper:rib/rib-table-ids/rib-table-id/information
  sensor-path Cisco-IOS-XR-ip-rib-ipv4-oper:rib/vrfs/vrf/afs/af/safs/saf/ip-rib-route-table-names/ip-rib-route-table-name/routes/route
 !
 sensor-group INT_STATS
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/cache/data-rate/input-data-rate
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/cache/data-rate/output-data-rate
 !
 subscription ISIS
  sensor-group-id ISIS sample-interval 30000
  destination-id DGroup2
 !
  subscription ISIS
  sensor-group-id ISIS sample-interval 30000
  destination-id DGroup2
 !
 subscription Sub2
  sensor-group-id SGroup2 sample-interval 30000
  destination-id DGroup2
 !
 subscription MPLSTE
  sensor-group-id MPLSTE sample-interval 60000
  destination-id DGroup2
 !
 subscription ipv4_bgp
  sensor-group-id ipv4_bgp sample-interval 30000
  destination-id DGroup2
 !        
 subscription rib-ipv4
  sensor-group-id rib-ipv4 sample-interval 30000
  destination-id DGroup2
 !
 subscription INT_STATS
  sensor-group-id INT_STATS sample-interval 30000
  destination-id DGroup2
 !
!
```
I have setup a bunch of things around system parameters, control plane and data plane. Please refer to [Cisco Feature Navigator](https://cfnng.cisco.com/ios-xr/yang-explorer/view-data-model) to get the YANG models as per your requirement. 

### Checks on IOS-XR
<pre>
RP/0/RP0/CPU0:PE1#show telemetry model-driven summary 
Tue Dec 31 07:23:58.009 UTC
 Subscriptions         Total:    6      <b>Active:    6</b>       Paused:    0
 Destination Groups    Total:    1
 Destinations       grpc-tls:    0 grpc-nontls:    1          tcp:    0            udp:    0
                      dialin:    0      Active:    1     <b>Sessions:    6</b>     Connecting:    0
 Sensor Groups         Total:    6
 Num of Unique Sensor Paths :   12
 Sensor Paths          Total:   12      Active:   12 Not Resolved:    0
 Max Sensor Paths           : 1000
 Max Containers per path    :   16
 Minimum target defined cadence :   30000
 Target Defined cadence factor  :    2
RP/0/RP0/CPU0:PE1#
</pre>

Looking at one particular subscription
<pre>
RP/0/RP0/CPU0:PE1#show telemetry model-driven subscription Sub2 internal 
Tue Dec 31 07:26:47.149 UTC
Subscription:  Sub2
-------------
  State:       ACTIVE
  Sensor groups:
  Id: SGroup2
    Sample Interval:      30000 ms
    Heartbeat Interval:   NA
    Sensor Path:          Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring/cpu-utilization
    <b>Sensor Path State:    Resolved</b>
    Sensor Path:          Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary
    Sensor Path State:    Resolved

  Destination Groups:
  Group Id: DGroup2
    Destination IP:       192.168.100.1
    Destination Port:     57000
    Encoding:             self-describing-gpb
    Transport:            grpc
    State:                Active
    TLS :                 False
    <b>Total bytes sent:     4140731</b>
    <b>Total packets sent:   73</b>
    Last Sent time:       2024-12-31 07:26:47.3658113843 +0000
          
  Collection Groups:
  ------------------
    Id: 4
    Sample Interval:      30000 ms
    Heartbeat Interval:   NA
    Heartbeat always:     False
    Encoding:             self-describing-gpb
    <b>Num of collection:    5</b>
    Incremental updates:  0
    Collection time:      Min:   201 ms Max:   328 ms
    Total time:           Min:   205 ms Avg:   267 ms Max:   372 ms
    Total Deferred:       0
    Total Send Errors:    0
    Total Send Drops:     0
    Total Other Errors:   0
    No data Instances:    0
    Last Collection Start:2024-12-31 07:26:47.3657951022 +0000
    Last Collection End:  2024-12-31 07:26:17.3628194656 +0000
    Sensor Path:          Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring/cpu-utilization

      Sysdb Path:     /oper/wdsysmon_fd/gl/*
      Count:          5 Method: DATALIST Min: 201 ms Avg: 237 ms Max: 328 ms
      Item Count:     11 Status: Active
      Missed Collections:0  send bytes: 3376266 packets: 15 dropped bytes: 0
      Missed Heartbeats: 0  Filtered Item Count: 0
                      success         errors          deferred/drops  
      Gets            0               0               
      List            0               0               
      Datalist        6               0               
      Finddata        0               0               
      GetBulk         0               0               
      Encode                          0               0               
      Send                            0               0               

    Id: 5
    Sample Interval:      30000 ms
    Heartbeat Interval:   NA
    Heartbeat always:     False
    Encoding:             self-describing-gpb
    Num of collection:    5
    Incremental updates:  0
    Collection time:      Min:    10 ms Max:    45 ms
    Total time:           Min:    13 ms Avg:    25 ms Max:    48 ms
    Total Deferred:       0
    Total Send Errors:    0
    Total Send Drops:     0
    Total Other Errors:   0
    No data Instances:    0
    Last Collection Start:2024-12-31 07:26:17.3628195821 +0000
    Last Collection End:  2024-12-31 07:26:17.3628216972 +0000
    Sensor Path:          Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary

      Sysdb Path:     /oper/showd/node/*/show_mem
      Count:          5 Method: GET Min: 10 ms Avg: 23 ms Max: 45 ms
      Item Count:     11 Status: Active
      Missed Collections:0  send bytes: 3310 packets: 5 dropped bytes: 0
      Missed Heartbeats: 0  Filtered Item Count: 0
                      success         errors          deferred/drops  
      Gets            11              0               
      List            6               0               
      Datalist        0               0               
      Finddata        6               0               
      GetBulk         0               0               
      Encode                          0               0               
      Send                            0               0               

RP/0/RP0/CPU0:PE1#
</pre>
Querying the InfluxDB
<pre>
<b>select "free_physical_memory","node_name" from "Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary"</b>

Interactive Table View (press q to exit mode, shift+up/down to navigate tables):
Name: Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary
┏━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━┓
┃ index ┃           time           ┃ free_physical_memory  ┃ node_name  ┃
┣━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━━━━━━━━━━━━╋━━━━━━━━━━━━┫
┃      1┃2024-12-30T04:28:52.419Z  ┃   652320768.0000000000┃0/RP0/CPU0  ┃
┃      2┃2024-12-30T04:28:52.423Z  ┃  2699067392.0000000000┃0/0/CPU0    ┃
┃      3┃2024-12-30T04:29:22.373Z  ┃   651173888.0000000000┃0/RP0/CPU0  ┃
┃      4┃2024-12-30T04:29:22.377Z  ┃  2699079680.0000000000┃0/0/CPU0    ┃
┃      5┃2024-12-30T04:29:52.357Z  ┃   650625024.0000000000┃0/RP0/CPU0  ┃
┃      6┃2024-12-30T04:29:52.361Z  ┃  2699071488.0000000000┃0/0/CPU0    ┃
┃      7┃2024-12-30T04:30:22.341Z  ┃   648966144.0000000000┃0/RP0/CPU0  ┃
┃      8┃2024-12-30T04:30:22.346Z  ┃  2696978432.0000000000┃0/0/CPU0    ┃
┃      9┃2024-12-30T04:30:52.431Z  ┃   647417856.0000000000┃0/RP0/CPU0  ┃
┃     10┃2024-12-30T04:30:52.441Z  ┃  2694713344.0000000000┃0/0/CPU0    ┃
┃     11┃2024-12-30T04:31:22.355Z  ┃   647303168.0000000000┃0/RP0/CPU0  ┃
┃     12┃2024-12-30T04:31:22.371Z  ┃  2694729728.0000000000┃0/0/CPU0    ┃
┃     13┃2024-12-30T04:31:52.344Z  ┃   647397376.0000000000┃0/RP0/CPU0  ┃
┃     14┃2024-12-30T04:31:52.347Z  ┃  2694721536.0000000000┃0/0/CPU0    ┃
┃     15┃2024-12-30T04:32:22.334Z  ┃   647389184.0000000000┃0/RP0/CPU0  ┃
┃     16┃2024-12-30T04:32:22.339Z  ┃  2694733824.0000000000┃0/0/CPU0    ┃
┃     17┃2024-12-30T04:32:52.356Z  ┃   645771264.0000000000┃0/RP0/CPU0  ┃
┃     18┃2024-12-30T04:32:52.361Z  ┃  2694598656.0000000000┃0/0/CPU0    ┃
┃     19┃2024-12-30T04:33:22.357Z  ┃   645820416.0000000000┃0/RP0/CPU0  ┃
┃     20┃2024-12-30T04:33:22.361Z  ┃  2694590464.0000000000┃0/0/CPU0    ┃
┃     21┃2024-12-30T04:33:52.39Z   ┃   645697536.0000000000┃0/RP0/CPU0  ┃
┃     22┃2024-12-30T04:33:52.394Z  ┃  2694574080.0000000000┃0/0/CPU0    ┃
┃     23┃2024-12-30T04:34:22.392Z  ┃   645849088.0000000000┃0/RP0/CPU0  ┃
┃     24┃2024-12-30T04:34:22.396Z  ┃  2702364672.0000000000┃0/0/CPU0    ┃
┃     25┃2024-12-30T04:34:52.364Z  ┃   645808128.0000000000┃0/RP0/CPU0  ┃
┃     26┃2024-12-30T04:34:52.369Z  ┃  2693156864.0000000000┃0/0/CPU0    ┃
┣━━━━━━━┻━━━━━━━━━━━━━━━━━━━━━━━━━━┻━━━━━━━━━━━━━━━━━━━━━━━┻━━━━━━━━━━━━┫
┃                                       4 Columns, 5942 Rows, Page 1/229┃
┃                                               Table 1/1, Statement 1/1┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
</pre>

Equivalent query for Grafana in Flex Language Syntax
```
from(bucket: "mdtlab")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r["_measurement"] == "Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary")
  |> filter(fn: (r) => r["_field"] == "free_physical_memory")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
  |> yield(name: "mean")
```

Personally, I found the Flex Language syntax a bit tricky to write from scratch, thankfully, we can use `Query Builder` option in InfluxDB GUI to get the appropriate Flex Language query for your needs.

Finally, it all comes together in Grafana dashboard.
![Grafana](/images/grafana_1.png)

### Troubleshooting

It's all well and good when things work fine but when they don't, we need to know what logs/traces to check to narrow down the issue.

I have simulated a failure scenario, due to which none of the sessions are `Active`, all are stuck in `Connecting` state.
<pre>
RP/0/RP0/CPU0:PE1#show telemetry model-driven subscription Sub2 internal 
Tue Dec 31 07:56:34.848 UTC
Subscription:  Sub2RP/0/RP0/CPU0:PE1#show telemetry model-driven summary 
Tue Dec 31 07:56:00.701 UTC
 Subscriptions         Total:    6      <b>Active:    0</b>       Paused:    0
 Destination Groups    Total:    1
 Destinations       grpc-tls:    0 grpc-nontls:    1          tcp:    0            udp:    0
                      dialin:    0      Active:    0     Sessions:    0     <b>Connecting:    6</b>
 Sensor Groups         Total:    6
 Num of Unique Sensor Paths :   12
 Sensor Paths          Total:   12      Active:    0 Not Resolved:    0
 Max Sensor Paths           : 1000
 Max Containers per path    :   16
 Minimum target defined cadence :   30000
 Target Defined cadence factor  :    2
</pre>

Checking one particular Subscription, there are no active collection groups.
<pre>
RP/0/RP0/CPU0:PE1#

-------------
  State:       NA
  Sensor groups:
  Id: SGroup2
    Sample Interval:      30000 ms
    Heartbeat Interval:   NA
    Sensor Path:          Cisco-IOS-XR-wdsysmon-fd-oper:system-monitoring/cpu-utilization
    Sensor Path State:    Resolved
    Sensor Path:          Cisco-IOS-XR-nto-misc-oper:memory-summary/nodes/node/summary
    Sensor Path State:    Resolved

  Destination Groups:
  Group Id: DGroup2
    Destination IP:       192.168.100.1
    Destination Port:     57000
    Encoding:             self-describing-gpb
    Transport:            grpc
    State:                NA
    TLS :                 False

  Collection Groups:
  ------------------
  <b>No active collection groups</b>

RP/0/RP0/CPU0:PE1#
</pre>

Checking the telemetry model-driven traces, we can see that TCP Connection is failing to Establish.
```
RP/0/RP0/CPU0:PE1#show telemetry model-driven trace all reverse 
Tue Dec 31 07:59:49.160 UTC
24118 wrapping entries (26496 possible, 24704 allocated, 0 filtered, 351346 total)
Dec 31 07:59:44.446 m2m/mdt/go-info 0/RP0/CPU0 t31604  5469 [mdt_go_trace_info]: mdtDialer:279 1: Reuse dialerChan 0xc000010218 for default
Dec 31 07:59:44.446 m2m/mdt/go-info 0/RP0/CPU0 t31604  5468 [mdt_go_trace_info]: getAddDialerChanWithLock:153 1:Found existing dialer for namespace default
Dec 31 07:59:44.446 m2m/mdt/go-info 0/RP0/CPU0 t31604  5467 [mdt_go_trace_info]: mdtDialer:269 1: namespace default, args 192.168.100.1:57000
Dec 31 07:59:44.445 m2m/mdt/go-info 0/RP0/CPU0 t31604  5466 [mdt_go_trace_info]: mdtConnEstablish:268 dial: target 192.168.100.1:57000
Dec 31 07:59:44.445 m2m/mdt/go-info 0/RP0/CPU0 t31604  5465 [mdt_go_trace_info]: mdtConnEstablish:229 1: req 503, TLS is false, cert , host , compression .
Dec 31 07:59:44.445 m2m/mdt/go-info 0/RP0/CPU0 t31604  5464 [mdt_go_trace_info]: mdtConnEstablish:229 1: Dialing out to 192.168.100.1:57000, req 503
Dec 31 07:59:44.445 m2m/mdt/go-info 0/RP0/CPU0 t31604  5463 [mdt_go_trace_info]: dialReqAgent:489 1: req 501, entries when dequeue is 3
Dec 31 07:59:44.445 m2m/mdt/subdb 0/RP0/CPU0 t31604  5462 [mdt_conn_grpc_establish_done]: Failed to establish conn 0
Dec 31 07:59:44.445 m2m/mdt/subdb 0/RP0/CPU0 t31604  5461 [mdt_conn_grpc_establish_done]: Got grpc establish callback 501, chan 18446744073709551615, status 1
Dec 31 07:59:44.445 m2m/mdt/go-info 0/RP0/CPU0 t31604  5460 [mdt_go_trace_error]: mdtConnEstablish:295 1: grpc service call failed, ReqId 501, 192.168.100.1:57000, rpc error: code =
 Unavailable desc = connection error: desc = "transport: error while dialing: dial tcp 192.168.100.1:57000: i/o timeout"
```

grpc traces also reveal the same.
```
RP/0/RP0/CPU0:PE1#show grpc trace all reverse | i 07:59:44
Tue Dec 31 08:37:28.700 UTC
Dec 31 07:59:44.446 ems/grpc 0/RP0/CPU0 t31542 EMS-GRPC: [core] Channel Connectivity change to CONNECTING
Dec 31 07:59:44.446 ems/grpc 0/RP0/CPU0 t31542 EMS-GRPC: [core] pickfirstBalancer: UpdateSubConnState: 0xc00001e8e0, {CONNECTING <nil>}
Dec 31 07:59:44.446 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] Subchannel picks a new address "192.168.100.1:57000" to connect
Dec 31 07:59:44.446 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] Subchannel Connectivity change to CONNECTING
Dec 31 07:59:44.446 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] blockingPicker: the picked transport is not ready, loop back to repick
Dec 31 07:59:44.446 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] Channel switches to new LB policy "pick_first"
Dec 31 07:59:44.446 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] ClientConn switching balancer to "pick_first"
Dec 31 07:59:44.446 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] ccResolverWrapper: sending update to cc: {[{192.168.100.1:57000  <nil> 0 <nil>}] <nil> <nil>}
Dec 31 07:59:44.445 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] scheme "" not registered, fallback to default scheme
Dec 31 07:59:44.445 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] parsed scheme: ""
Dec 31 07:59:44.445 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] Subchannel Connectivity change to SHUTDOWN
Dec 31 07:59:44.445 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] Channel Connectivity change to SHUTDOWN
Dec 31 07:59:44.445 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] Channel Connectivity change to TRANSIENT_FAILURE
Dec 31 07:59:44.445 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] pickfirstBalancer: UpdateSubConnState: 0xc000019fb0, {TRANSIENT_FAILURE connection error: desc = "transport: error wh
ile dialing: dial tcp 192.168.100.1:57000: i/o timeout"}
Dec 31 07:59:44.445 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] Subchannel Connectivity change to TRANSIENT_FAILURE
Dec 31 07:59:44.445 ems/grpc 0/RP0/CPU0 t31604 EMS-GRPC: [core] grpc: addrConn.createTransport failed to connect to {192.168.100.1:57000 192.168.100.1:57000 <nil> 0 <nil>}. Err: con
nection error: desc = "transport: error while dialing: dial tcp 192.168.100.1:57000: i/o timeout"
RP/0/RP0/CPU0:PE1#
```

We can also check the TCP connection state.
```
RP/0/RP0/CPU0:PE1#bash netstat -an
Tue Dec 31 09:27:56.895 UTC
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      1 211.0.0.9:61390         192.168.100.1:57000     SYN_SENT   
tcp6       0      0 :::57400                :::*                    LISTEN     
Active UNIX domain sockets (servers and established)
Proto RefCnt Flags       Type       State         I-Node   Path
unix  2      [ ACC ]     STREAM     LISTENING     14646215 /var/lib/malloc_stat/9580
unix  2      [ ]         DGRAM                    14646222 

RP/0/RP0/CPU0:PE1#
```

In this particular case, server didn't have a route to Router's Loopback, as soon I added the route, Connections came up.

```
RP/0/RP0/CPU0:PE1#show grpc trace all reverse 
Tue Dec 31 08:50:05.358 UTC
83163 wrapping entries (549888 possible, 328448 allocated, 0 filtered, 149016 total)
Dec 31 08:48:04.729 ems/grpc 0/RP0/CPU0 t31542 EMS-GRPC: [core] Channel Connectivity change to READY
Dec 31 08:48:04.729 ems/grpc 0/RP0/CPU0 t31542 EMS-GRPC: [core] pickfirstBalancer: UpdateSubConnState: 0xc0001646c0, {READY <nil>}
Dec 31 08:48:04.729 ems/grpc 0/RP0/CPU0 t31542 EMS-GRPC: [core] Subchannel Connectivity change to READY
Dec 31 08:48:04.709 ems/grpc 0/RP0/CPU0 t31542 EMS-GRPC: [core] Channel Connectivity change to CONNECTING
Dec 31 08:48:04.709 ems/grpc 0/RP0/CPU0 t31542 EMS-GRPC: [core] pickfirstBalancer: UpdateSubConnState: 0xc0001646c0, {CONNECTING <nil>}
Dec 31 08:48:04.708 ems/grpc 0/RP0/CPU0 t31659 EMS-GRPC: [core] Subchannel picks a new address "192.168.100.1:57000" to connect
Dec 31 08:48:04.708 ems/grpc 0/RP0/CPU0 t31659 EMS-GRPC: [core] Subchannel Connectivity change to CONNECTING
Dec 31 08:48:04.708 ems/grpc 0/RP0/CPU0 t31659 EMS-GRPC: [core] blockingPicker: the picked transport is not ready, loop back to repick
Dec 31 08:48:04.708 ems/grpc 0/RP0/CPU0 t31659 EMS-GRPC: [core] Channel switches to new LB policy "pick_first"
Dec 31 08:48:04.708 ems/grpc 0/RP0/CPU0 t31659 EMS-GRPC: [core] ClientConn switching balancer to "pick_first"
Dec 31 08:48:04.708 ems/grpc 0/RP0/CPU0 t31659 EMS-GRPC: [core] ccResolverWrapper: sending update to cc: {[{192.168.100.1:57000  <nil> 0 <nil>}] <nil> <nil>}

RP/0/RP0/CPU0:PE1#bash netstat -an
Tue Dec 31 08:51:09.756 UTC
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 211.0.0.9:60665         192.168.100.1:57000     ESTABLISHED
tcp        0      0 211.0.0.9:60658         192.168.100.1:57000     ESTABLISHED
tcp        0      0 211.0.0.9:60659         192.168.100.1:57000     ESTABLISHED
tcp        0      0 211.0.0.9:60668         192.168.100.1:57000     ESTABLISHED
tcp        0      0 211.0.0.9:60661         192.168.100.1:57000     ESTABLISHED
tcp        0      0 211.0.0.9:60660         192.168.100.1:57000     ESTABLISHED
tcp6       0      0 :::57400                :::*                    LISTEN     
Active UNIX domain sockets (servers and established)
Proto RefCnt Flags       Type       State         I-Node   Path
unix  2      [ ACC ]     STREAM     LISTENING     13474827 /var/lib/malloc_stat/2458
unix  2      [ ]         DGRAM                    13474834 

RP/0/RP0/CPU0:PE1#
```

### Validating YANG Data model using in-built Python script

IOS-XR comes with a handy in-built python script, `mdt_exec.py` which can be invoked to quickly check the YANG model for any particular path.

```
RP/0/RP0/CPU0:PE1#run mdt_exec.py -s Cisco-IOS-XR-clns-isis-oper:isis/instance$
Tue Dec 31 08:56:16.248 UTC
{
  "encoding_path": "Cisco-IOS-XR-clns-isis-oper:isis/instances/instance/summary", 
  "subscription_id_str": "app_TEST_200000001", 
  "collection_start_time": "1735635376550", 
  "msg_timestamp": "1735635376562", 
  "collection_end_time": "1735635376562", 
  "node_id_str": "PE1", 
  "data_json": [
    {
      "keys": [
        {
          "instance-name": "1"
        }
      ], 
      "timestamp": "1735635376561", 
      "content": {
        "l2-adjacencies": 3, 
        "l1-routers": 0, 
        "l1ls-ps": 0, 
        "l2ls-ps": 5, 
        "l2-routers": 5, 
        "running-levels": "isis-levels-2", 
        "l1-adjacencies": 0, 
        "l1l2-adjacencies": 0
      }
    }
  ], 
  "collection_id": "185"
},
^CDone
```

This was meerly scratching the surface but it was great fun setting everything up and to finally see those shinny graphs. Cisco has some very good documentation on this subject matter, please check them out if you're thinking of setting this up at home or at work.

### Reference

https://xrdocs.io/telemetry/

https://xrdocs.io/telemetry/tutorials/



