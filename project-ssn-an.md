## Project SSN + AN

sources:

1. https://wiki.opendaylight.org/view/OpenDaylight_OpenFlow_Plugin:_TLS_Support
2. https://ryu.readthedocs.io/en/latest/tls.html



## Useful bits

When one interface is connected to cloud, but packets get send via another interface (that's not connected) you can shut the annoying interface off with command below:

```
ip link set dev eth0 down
```



## Setting up the controller and the switch

Install python3

```
apt-get install python3
```



Install pip

```
apt-get install python3-pip
```



On controller install Ryu (make sure to use version 9.4 of `tinyrpc` package)

```
pip3 install ryu
```



On each switch check what we can install:

````
$ apt-cache search openvswitch
neutron-common - OpenStack virtual network service - common files
neutron-openvswitch-agent - OpenStack virtual network service - Open vSwitch agent
neutron-taas-openvswitch-agent - OpenStack virtual network service - Tap-as-a-Service agent
openstack-proxy-node - Metapackage to install an Openstack proxy node
openvswitch-common - Open vSwitch common components
openvswitch-dbg - Debug symbols for Open vSwitch packages
openvswitch-dev - Open vSwitch development package
openvswitch-pki - Open vSwitch public key infrastructure dependency package
openvswitch-switch - Open vSwitch switch implementations
openvswitch-testcontroller - Simple controller for testing OpenFlow setups
openvswitch-vtep - Open vSwitch VTEP utilities
python-openvswitch - Python bindings for Open vSwitch
python3-openvswitch - Python 3 bindings for Open vSwitch 
````



Install on each switch as below:

```
apt-get install openvswitch-switch openvswitch-pki python3-openvswitch
```

