# AN Lab 3 - SDN

#### Artem Abramov SNE19

I decided to make a PDF document instead of a video (as was suggested), because I wanted to have a reference for future setups.

## 1. Preparation

### Virtual routing solution

I will be using Open vSwitch, for the SDN controller I will use Ryu (http://osrg.github.io/ryu/)

### Network scheme

My network scheme is shown below:

![an-lab-3-sdn - GNS3_509](AN-Lab-3-sdn.assets/an-lab-3-sdn%20-%20GNS3_509.png)



## 2. Deployment

The IP addresses are configured as shown in the diagram above. It is necessary to configure an IP address on the OpenvSwitch as well, otherwise how would it receive responce packets from the controller.

The first step is configuring OpenvSwitch, telling it where the SDN controller can be found:
```
/ # ovs-vsctl set-controller br0 tcp:10.0.0.1:6653
```

The next step is configuring the machine that will act as the SDN controller. The steps below were done on the `ryu-controller` machine. Installing Ryu is shown below:
```
# pip2 install ryu
```

Ryu depends on the tinyrpc package, however since version 1.x.x tinyrpc only supports python 3, therefore I had to remove tinyrpc and reinstall an older version 0.9.4 as shown below (source: https://pypi.org/project/tinyrpc/#description):
```
# pip2 uninstall tinyrpc
# pip2 install tinyrpc==0.9.4
```

Then I choose a simple example application called `simple_switch.py` and run it with ryu:
```
# ryu-manager simple_switch.py
loading app simple_switch.py
loading app ryu.controller.ofp_handler
instantiating app simple_switch.py of SimpleSwitch
instantiating app ryu.controller.ofp_handler of OFPHandler
packet in 69244875445583 00:23:20:da:82:6f ff:ff:ff:ff:ff:ff 65534
```

This means that the controller is running and a slave switch was found. Now we can test it by pinging from the user to the ryu-controller as shown below:

![QEMU (AN-Lab-3-sdn.assets/QEMU%20(user)%20-%20TigerVNC_510.png) - TigerVNC_510](../../../Pictures/QEMU%20(user)%20-%20TigerVNC_510.png)



The following output appeared from `ryu-manageer`:
```
root@kali:~/dataryu/app# ryu-manager simple_switch.py
loading app simple_switch.py
loading app ryu.controller.ofp_handler
instantiating app simple_switch.py of SimpleSwitch
instantiating app ryu.controller.ofp_handler of OFPHandler
packet in 69244875445583 00:23:20:da:82:6f ff:ff:ff:ff:ff:ff 65534
packet in 69244875445583 0c:68:04:67:d8:06 0c:68:04:dd:55:07 3
packet in 69244875445583 0c:68:04:dd:55:07 0c:68:04:67:d8:06 2
packet in 69244875445583 0c:68:04:67:d8:06 0c:68:04:dd:55:07 3
packet in 69244875445583 3e:fa:54:34:15:4f 33:33:00:00:00:02 65534
```

And the table in OpenvSwitch showed the info as below:
```
/ # ovs-ofctl  dump-flows br0
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=179.723s, table=0, n_packets=8, n_bytes=670, idle_age=67, in_port=2,dl_src=0c:68:04:dd:55:07,dl_dst=0c:68:04:67:d8:06 actions=output:3
 cookie=0x0, duration=178.723s, table=0, n_packets=18, n_bytes=1320, idle_age=52, in_port=3,dl_src=0c:68:04:67:d8:06,dl_dst=0c:68:04:dd:55:07 actions=output:2
```



The MAC `0c:68:04:dd:55:07` belongs to machine with IP `10.0.0.1` and MAC `0c:68:04:67:d8:06` belongs to the user machine with IP `10.0.0.2`.



Then I slightly expanded my topology as shown below:

![an-lab-3-sdn - GNS3_511](AN-Lab-3-sdn.assets/an-lab-3-sdn%20-%20GNS3_511.png)

After adding the new switch and pinging from `10.0.0.5` to `10.0.0.1` the flow table at OpenvSwitch-3 (ip `10.0.0.3`) has changed as shown below:

```
/ # ovs-ofctl  dump-flows br0
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=402.528s, table=0, n_packets=0, n_bytes=0, idle_age=402, in_port=2,dl_src=0c:68:04:dd:55:07,dl_dst=82:ad:3c:58:97:1c actions=output:6
 cookie=0x0, duration=399.844s, table=0, n_packets=172, n_bytes=13675, idle_age=4, in_port=3,dl_src=0c:68:04:67:d8:06,dl_dst=0c:68:04:dd:55:07 actions=output:2
 cookie=0x0, duration=379.696s, table=0, n_packets=10, n_bytes=600, idle_age=29, in_port=2,dl_src=0c:68:04:dd:55:07,dl_dst=0c:68:04:67:d8:06 actions=output:3
 cookie=0x0, duration=6.464s, table=0, n_packets=3, n_bytes=294, idle_age=4, in_port=2,dl_src=0c:68:04:dd:55:07,dl_dst=00:50:79:66:68:00 actions=output:6
 cookie=0x0, duration=6.456s, table=0, n_packets=2, n_bytes=196, idle_age=4, in_port=6,dl_src=00:50:79:66:68:00,dl_dst=0c:68:04:dd:55:07 actions=output:2
```

There are three additional entries:

1. Communication from ryu-controller to MAC `82:ad:3c:58:97:1c` which is OpenvSwitch-4
2. Communication between VPCS with ip `10.0.0.5` to ryu-controller.
3. Communication from ryu-controller back to the VPCS.



We can see on the screenshot below that the version of OpenFlow used to distribute the rules is `1.0`:

![Capturing from Standard input [ruy-controller Ethernet7 to OpenvSwitch-3 eth1]_512](AN-Lab-3-sdn.assets/Capturing%20from%20Standard%20input%20%5Bruy-controller%20Ethernet7%20to%20OpenvSwitch-3%20eth1%5D_512.png)

Now is a good time to try another ryu app that would somehow alter the network. Currently the simple_switch.py app is running which tells the network hardware to function as standard switches.

I decided to try configuring MPLS. I found an example application on gihub: https://github.com/pidEins/MPLS-RYU-CONTROLLER-APPLICATION

The repository has some very good resources detailing how the MPLS app was created. However the actual app fails with errors. After spending the whole day trying to fix the errors I could finally start the MPLS app as shown below:

```
# ryu-manager --verbose rest_mpls.py
> /usr/local/lib/python2.7/dist-packages/ryu/utils.py(110)import_module()
-> return importlib.import_module(modname)
(Pdb) c
loading app rest_mpls.py
loading app ryu.controller.ofp_handler
> /usr/local/lib/python2.7/dist-packages/ryu/utils.py(110)import_module()
-> return importlib.import_module(modname)
(Pdb) c
instantiating app None of DPSet
creating context dpset
creating context wsgi
instantiating app rest_mpls.py of RestRouterAPI
instantiating app ryu.controller.ofp_handler of OFPHandler
BRICK dpset
  PROVIDES EventDP TO {'RestRouterAPI': set(['dpset'])}
  CONSUMES EventOFPSwitchFeatures
  CONSUMES EventOFPPortStatus
  CONSUMES EventOFPStateChange
BRICK RestRouterAPI
  CONSUMES EventOFPPacketIn
  CONSUMES EventDP
  CONSUMES EventOFPStatsReply
  CONSUMES EventOFPFlowStatsReply
BRICK ofp_event
  PROVIDES EventOFPPortStatus TO {'dpset': set(['main'])}
  PROVIDES EventOFPStateChange TO {'dpset': set(['main', 'dead'])}
  PROVIDES EventOFPSwitchFeatures TO {'dpset': set(['config'])}
  PROVIDES EventOFPStatsReply TO {'RestRouterAPI': set(['main'])}
  PROVIDES EventOFPFlowStatsReply TO {'RestRouterAPI': set(['main'])}
  PROVIDES EventOFPPacketIn TO {'RestRouterAPI': set(['main'])}
  CONSUMES EventOFPErrorMsg
  CONSUMES EventOFPPortStatus
  CONSUMES EventOFPEchoRequest
  CONSUMES EventOFPSwitchFeatures
  CONSUMES EventOFPEchoReply
  CONSUMES EventOFPPortDescStatsReply
  CONSUMES EventOFPHello
(2886) wsgi starting up on http://0.0.0.0:8080
connected socket:<eventlet.greenio.base.GreenSocket object at 0x7ffbb091ffd0> address:('10.0.0.3', 54366)
hello ev <ryu.controller.ofp_event.EventOFPHello object at 0x7ffbb0929650>
move onto config mode
connected socket:<eventlet.greenio.base.GreenSocket object at 0x7ffbb09292d0> address:('10.0.0.4', 39596)
hello ev <ryu.controller.ofp_event.EventOFPHello object at 0x7ffbb09299d0>
move onto config mode
EVENT ofp_event->dpset EventOFPSwitchFeatures
switch features ev version=0x4,msg_type=0x6,msg_len=0x20,xid=0xd9524884,OFPSwitchFeatures(auxiliary_id=0,capabilities=79,datapath_id=69244875445583,n_buffers=256,n_tables=254)
EVENT ofp_event->dpset EventOFPSwitchFeatures
switch features ev version=0x4,msg_type=0x6,msg_len=0x20,xid=0x2cef1c20,OFPSwitchFeatures(auxiliary_id=0,capabilities=79,datapath_id=7528964166476,n_buffers=256,n_tables=254)
move onto main mode
EVENT ofp_event->dpset EventOFPStateChange
DPSET: register datapath <ryu.controller.controller.Datapath object at 0x7ffbb0929090>
EVENT dpset->RestRouterAPI EventDP
move onto main mode
EVENT ofp_event->dpset EventOFPStateChange
[RT][INFO] switch_id=00003efa5434154f: Set SW config for TTL error packet in.
------------ yyd tunnel_id= 0
-------- yyd OfCtl_after_v1_2 set_flow
[RT][INFO] switch_id=00003efa5434154f: Set ARP handling (packet in) flow [cookie=0x0]
-------- yyd OfCtl_after_v1_2 set_flow
[RT][INFO] switch_id=00003efa5434154f: Set L2 switching (normal) flow [cookie=0x0]
------------ yyd tunnel_id= 0
-------- yyd OfCtl_after_v1_2 set_flow
[RT][INFO] switch_id=00003efa5434154f: Start cyclic routing table update.
[RT][INFO] switch_id=00003efa5434154f: Join as router.
DPSET: register datapath <ryu.controller.controller.Datapath object at 0x7ffbb0929290>
EVENT dpset->RestRouterAPI EventDP
[RT][INFO] switch_id=000006d8f93c134c: Set SW config for TTL error packet in.
------------ yyd tunnel_id= 0
-------- yyd OfCtl_after_v1_2 set_flow
[RT][INFO] switch_id=000006d8f93c134c: Set ARP handling (packet in) flow [cookie=0x0]
-------- yyd OfCtl_after_v1_2 set_flow
[RT][INFO] switch_id=000006d8f93c134c: Set L2 switching (normal) flow [cookie=0x0]
------------ yyd tunnel_id= 0
-------- yyd OfCtl_after_v1_2 set_flow
[RT][INFO] switch_id=000006d8f93c134c: Start cyclic routing table update.
[RT][INFO] switch_id=000006d8f93c134c: Join as router.
EVENT ofp_event->RestRouterAPI EventOFPPacketIn
EVENT ofp_event->RestRouterAPI EventOFPPacketIn
EVENT ofp_event->RestRouterAPI EventOFPPacketIn
EVENT ofp_event->RestRouterAPI EventOFPPacketIn
EVENT ofp_event->RestRouterAPI EventOFPPacketIn
EVENT ofp_event->RestRouterAPI EventOFPPacketIn
EVENT ofp_event->RestRouterAPI EventOFPPacketIn
EVENT ofp_event->RestRouterAPI EventOFPPacketIn
```



As shown in the wireshark capture below, the MPLS app uses OpenFlow 1.3:

![Capturing from Standard input [OpenvSwitch-4 eth2 to OpenvSwitch-3 eth5]_513](AN-Lab-3-sdn.assets/Capturing%20from%20Standard%20input%20%5BOpenvSwitch-4%20eth2%20to%20OpenvSwitch-3%20eth5%5D_513.png)



Then I realized that for MPLS I had to configure the OpenvSwitches as routers, not as switches. For this I configured an ip address on each switch. Unfortunately further configuration was too difficult because of the errors present in the MPLS app that prevented distribution of labels. Unfortunately I could not fix the errors on time.

