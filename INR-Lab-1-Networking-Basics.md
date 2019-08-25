# INR Lab 1 - Networking Basics
#### Artem Abramov SNE19

## Task 1 - Tools

### 1. Install the needed dependencies for GNS3: QEMU/KVM , Docker , and Wireshark

I checked that virtualisation is enabled in bios with command:  
```
$ cat /proc/cpuinfo | egrep 'vmx'  
```
source: https://www.cyberciti.biz/faq/linux-xen-vmware-kvm-intel-vt-amd-v-support/

Then was installing wireshark. During the install process there was a choice to create the special wireshark group or not. I choose not to create it.   
```
$ sudo apt-get install wireshark-qt wieshark-doc 
```

Later I changed my mind and re-configured wireshark to create a special wireshark group by following this guide: https://code.wireshark.org/review/gitweb?p=wireshark.git;a=blob_plain;f=debian/README.Debian;hb=HEAD

Docker was installed using official documentation https://docs.docker.com/install/linux/docker-ce/ubuntu/ : 

Update the apt package index:  
```
$ sudo apt-get update
```

Install packages to allow apt to use a repository over HTTPS:
```
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```
Add Docker’s official GPG key:
```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

Use the following command to set up the stable repository:
```
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```
Update the apt package index again:  
```
$ sudo apt-get update
```

Install the latest version of Docker Engine - Community and containerd: 
```
$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```

GNS3 was installed without installing QEMU separately, by following the instructions in the article for GNS3:
https://docs.gns3.com/1QXVIihk7dsOL7Xr7Bmz4zRzTsJ02wklfImGuHwTlaA4/index.html
```
$ sudo add-apt-repository ppa:gns3/ppa
$ sudo apt-get update
$ sudo apt-get install gns3-gui
```

According to the article to enable GNS3 to use docker my account must be added to the docker group:  
```
$ sudo usermod -a -G docker artem
```

Then I realised that I should probably install qemu-kvm 
separately and make sure that KVM is enabled.
I did this by following the guide at https://www.cyberciti.biz/faq/installing-kvm-on-ubuntu-16-04-lts-server/
```
sudo apt-get install qemu-kvm libvirt-bin virtinst bridge-utils cpu-checker
```

After installation I checked kvm using the supplied tool:  
```
$ kvm-ok  
INFO: /dev/kvm exists  
KVM acceleration can be used 
```

Finally added my account to various groups such as ubridge, libvirt, kvm, wireshark. 

In the end the groups for my account were as follows: 
```
$ groups
artem adm cdrom sudo dip plugdev lpadmin sambashare kvm ubridge libvirt wireshark docker
```

### 2. Start a GNS3 Project

Summary of the GNS3 setup is below.
![](https://i.imgur.com/Q9nUeJS.png)

Giving the project a name.
![](https://i.imgur.com/hkrXkch.png)

Because the template for Ubuntu Cloud Guest was not pre-installed I had to download and configure it.  

The instructions I used were from the official GNS3 guide: https://docs.gns3.com/appliances/ubuntu-cloud.html

I download the appliance file: https://raw.githubusercontent.com/GNS3/gns3-registry/master/appliances/ubuntu-cloud.gns3a  

Then I imported the .gns3a file in GNS3, following the tutorial: https://docs.gns3.com/1_3RdgLWgfk4ylRr99htYZrGMoFlJcmKAAaUAc8x9Ph8/index.html  

From then on I just followed the menus.
(In the process I had to download the .iso images for the supported Ubuntu version from this link https://docs.gns3.com/appliances/ubuntu-cloud.html#appliance_supported)

Screenshot showing possible versions of the Ubuntu Cloud Guest appliance.
![](https://i.imgur.com/nAZoNcm.png)

Screenshot of the configuration summary for Ubuntu Cloud Guest.
![](https://i.imgur.com/LAE44jD.png)

To get familiar with the GNS3 interface I used the getting started instructions from the GNS3 website. Two particularly useful links were:
1. Example connecting to the Internet: https://docs.gns3.com/1vFs-KENh2uUFfb47Q2oeSersmEK4WahzWX-HrMIMd00/index.html
2. Index of Getting Started documentation: https://docs.gns3.com/1FFbs5hOBbx8O855KxLetlCwlbymTN8L1zXXQzCqfmy4/index.html

Screenshot of the created appliance instance.
![](https://i.imgur.com/gEid5pc.png)

The final step was to check that the ubuntu guest can start.
The machine indeed booted successfully and presented a login prompt. Although it spend about 30 seconds waiting for a start job on boot, the exact job description was 'A start job is running for Wait for Network to be Configured (15s / no limit)'

Screenshot of the console prompt from ubuntu cloud guest.
![](https://i.imgur.com/SAchA6Q.png)

### 3. What are the different ways you can configure internet access in GNS3?

#### 1. Ethernet Cloud connection 

Easy to setup in GNS3, but it bypasses the TCP/IP stack of the host machine. This means that when the GNS3 guest is using the wire the host can not use it at the same time. This causes difficult to track down connectivity issues for both guest and host. (for example pinging google.com from the host and the guest at the same time is impossible. Ethernet cloud connection is uncomfourtable to use, because normally one  wants to access the network from the host and from the guest simultaneously).

First of all I changed the netplan configuration inside the ubuntu cloud guest to match the network setup of the host. Source: https://www.howtoforge.com/linux-basics-set-a-static-ip-on-ubuntu

Screenshot of the ubuntu cloud network config.
![](https://i.imgur.com/HFCoxG9.png)

After that the network still appeared to be broken. The next step to try was reverting the network configuration in GNS3 to legacy mode. This appeared to have fixed the issue.

Screenshot of the GNS3 network configuration for ubuntu guest.
![](https://i.imgur.com/WWvyndh.png)

Screenshot of successfull ping from inside the ubuntu guest.
![](https://i.imgur.com/huW3YuO.png)

Screenshot of the resulting topology created in GNS3 to test the Ethernet cloud connection.
![](https://i.imgur.com/weuAgBR.png)


#### 2. TAP Cloud connection

Tap device is used on the host to receive data from the guest. Quite easy to setup.

I created the TAP interface on the host using the NetworkManager CLI as described in this source: https://mail.gnome.org/archives/networkmanager-list/2016-January/msg00049.html

Screenshot showing the configuration of the TAP interface on the host.
![](https://i.imgur.com/ijlpzps.png)

To test this connection I created a new GNS3 project.
I was extremely  unsatisfied with UbuntuCloud guest, so I diverged from the rules and experimented with using AlpineLinux and KaliLinux as a guest. Eventually I setteled on the KaliLinux guest.
Therefore this type of connection was tested with the KaliLinux guest.

(I found KaliLinux guest to be by a long shot better than the Ubuntu guest.
First of all kali uses the familiar ifupdown and /etc/network/interfaces instead of netplan, second the kali appliance by default uses the VNC connection, I believe there are more benefits to be discovered.
A serious downside is that kali must be installed by hand on each machine in the lab, which takes time and harddrive space - 9.6GB for base install with X server.)

The next step was creating a kali guest in GNS3 and configuring its network and default gateway to suit the TAP interface on the host.

Screenshot of the network configuration on the kali guest.
![](https://i.imgur.com/1wjscuQ.png)

Screenshot of the topology created in GNS3.
![](https://i.imgur.com/bprmQBf.png)

Then I checked that packet forwarding was enabled on the host.
```
$ cat /proc/sys/net/ipv4/ip_forward
1
```
source: http://www.ducea.com/2006/08/01/how-to-enable-ip-forwarding-in-linux/

The last step was to set the host to act as a NAT for the traffic leaving the kali guest.
This was done with the command:
```
sudo iptables -t nat -A POSTROUTING -s 255.255.255.0/24 -o eth0 -j MASQUERADE
```
The command means take traffic with source ip in the 255.255.255.0/24 subnet and send it to eth0 interface after applying NAT.
Source: https://openvpn.net/community-resources/how-to/#routing-all-client-traffic-including-web-traffic-through-the-vpn.
Thus the traffic can go from the 255.255.255.0/24 subnet to the Internet and return. Communication from the host to the guest is also possible because of the last routing rule in the output:
```
$ ip route
default via 188.130.155.33 dev eno1 onlink 
169.254.0.0/16 dev eno1 scope link metric 1000 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
188.130.155.32/27 dev eno1 proto kernel scope link src 188.130.155.42 
192.168.122.0/24 dev virbr0 proto kernel scope link src 192.168.122.1 linkdown 
255.255.255.0/24 dev tap0 proto kernel scope link src 255.255.255.100 metric 450 
```

After the iptables rule was applied the kali guest could ping google.com and communication between the guest and the host was also possible.

Screenshot showing pings from the host to the kali guest, from the kali guest to the host and from the kali guest to google.com.
![](https://i.imgur.com/z8CP3CH.png)


#### 3. Linux Bridge and NAT Cloud connection 

This appears to be similar to how docker operates (https://www.securitynik.com/2016/12/docker-networking-internals-how-docker_16.html). To allow communication between guest and host a virtual bridge is used. Nat is used to allow connection to the internet. 

To setup the bridge on the host I followed two in depth articles:
1. Overview of linux bridging: https://cloudbuilder.in/blogs/2013/12/08/tap-interfaces-linux-bridge/
2. Creating a bridge: https://cloudbuilder.in/blogs/2013/12/02/linux-bridge-virtual-networking/

Decided to reuse the bridge that already existed on the host: virtbr0.
This bridge was created by installing libvirt. It can be configured
with
```
$ virsh net-edit default
```
source: https://www.bernhard-ehlers.de/blog/2017/07/11/gns3-cloud-linux.html

Just following along with the default configuration, as shown on the screenshot below:
![](https://i.imgur.com/15wDHuU.png)

The next step editing configuration of the GNS3 cloud to use virbr0 as its Ethernet interface. To be able to select the non-physical interface the box "Show special Ethernet interfaces" had to be ticked. Screenshot of configuration for the cloud is below.
![](https://i.imgur.com/pgazhp5.png)

After that connection inside the kali guest started working. 
The ip allocation on the side of kali guest was done automatically because libvirt provides DHCP.
The screenshot below shows that the kali guest connect to the virbr0 bridge by a newly created virtual interface "gns3tap0-2".
![](https://i.imgur.com/bg8exam.png)

The bridge does not have a physical interface connected to it, however traffic flow and NAT is handled by iptable rules.
```
$ sudo iptables -t nat -vnL POSTROUTING
Chain POSTROUTING (policy ACCEPT 17296 packets, 1089K bytes)
 pkts bytes target     prot opt in     out     source               destination         
   60  5281 RETURN     all  --  *      *       192.168.122.0/24     224.0.0.0/24        
    0     0 RETURN     all  --  *      *       192.168.122.0/24     255.255.255.255     
    3   180 MASQUERADE  tcp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    7   532 MASQUERADE  udp  --  *      *       192.168.122.0/24    !192.168.122.0/24     masq ports: 1024-65535
    2   168 MASQUERADE  all  --  *      *       192.168.122.0/24    !192.168.122.0/24 
```
source: https://stackoverflow.com/questions/37536687/what-is-the-relation-between-docker0-and-eth0

Screenshow of ping working in kali guest.
![](https://i.imgur.com/YVOFBS0.png)

Screenshot of the topology created in GNS3.
![](https://i.imgur.com/1yRojYi.png)

## Task 2 - Switching

To create this topology I used a pair of KaliLinux CLI guests (without X server installed).
The first step step was to setup the cloud connection
