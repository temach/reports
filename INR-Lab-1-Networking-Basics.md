# INR Lab 1 - Networking Basics
#### Artem Abramov SNE19

## Task 1 - Tools

### 1. Install the needed dependencies for GNS3: QEMU/KVM , Docker , and Wireshark

I checked that virtualisation is enabled in bios with command:  
$ cat /proc/cpuinfo | egrep 'vmx'  
[source: https://www.cyberciti.biz/faq/linux-xen-vmware-kvm-intel-vt-amd-v-support/]

#### Installing wireshark:
I had a choice to create the special wireshark group or not. I choose not to create it.   
$ sudo apt-get install wireshark-qt wieshark-doc 

Later I changed my mind and re-configured wireshark with this option by following the guide source: https://code.wireshark.org/review/gitweb?p=wireshark.git;a=blob_plain;f=debian/README.Debian;hb=HEAD

#### Installing docker
Using official documentation https://docs.docker.com/install/linux/docker-ce/ubuntu/ : 


Update the apt package index:  
$ sudo apt-get update

Install packages to allow apt to use a repository over HTTPS:
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

Add Docker’s official GPG key:
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

Use the following command to set up the stable repository:
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

Update the apt package index again:
$ sudo apt-get update

Install the latest version of Docker Engine - Community and containerd:
$ sudo apt-get install docker-ce docker-ce-cli containerd.io

#### Installing GNS3
Without installing QEMU separately, follow the instructions in the article for GNS3:
https://docs.gns3.com/1QXVIihk7dsOL7Xr7Bmz4zRzTsJ02wklfImGuHwTlaA4/index.html
installing GNS3 from the repository automatically installs QEMU.
$ sudo add-apt-repository ppa:gns3/ppa
$ sudo apt-get update
$ sudo apt-get install gns3-gui

#### Enable docker support
To enable docker support for GNS3 the user must b added to the docker group:
$ sudo usermod -a -G docker artem

#### Install QEMU/KVM separately
Then I realised that I should probably install qemu-kvm 
separately and make sure that KVM is enabled.
I did this by following the guide at https://www.cyberciti.biz/faq/installing-kvm-on-ubuntu-16-04-lts-server/

To check that kvm acceleration can be used:
$ kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used

#### Configure groups
Finally added my account to various groups:
ubridge libvirt kvm wireshark 

Current groups:
$ groups
artem adm cdrom sudo dip plugdev lpadmin sambashare kvm ubridge libvirt wireshark docker

### 2. Start a GNS3 Project

Setup GNS3 window.
![](https://i.imgur.com/Q9nUeJS.png)

Giving the project a name.
![](https://i.imgur.com/hkrXkch.png)

Because the template was not pre-installed I had to download and configure it.  

The instructions I used were: https://docs.gns3.com/appliances/ubuntu-cloud.html

Download the appliance file: https://raw.githubusercontent.com/GNS3/gns3-registry/master/appliances/ubuntu-cloud.gns3a  
Download the files for one of the supported version: https://docs.gns3.com/appliances/ubuntu-cloud.html#appliance_supported  
Import the .gns3a file in GNS3, following the tutorial: https://docs.gns3.com/1_3RdgLWgfk4ylRr99htYZrGMoFlJcmKAAaUAc8x9Ph8/index.html  

Configure the appliance.
![](https://i.imgur.com/nAZoNcm.png)

Configuration summary for Ubuntu Cloud Guest.
![](https://i.imgur.com/LAE44jD.png)


Then I created an appliance instance.
![](https://i.imgur.com/gEid5pc.png)


And checked that the machine can start, which was successfull.
Although it spend about 30 seconds waiting for a start job on boot,
the exact text was 'A start job is running for Wait for Network to be Configured (15s / no limit)'
![](https://i.imgur.com/SAchA6Q.png)


#### 3. What are the different ways you can configure internet access in GNS3?

To get familiar with the GNS3 interface I used the getting started insturcitons from the GNS3 website: https://docs.gns3.com/1vFs-KENh2uUFfb47Q2oeSersmEK4WahzWX-HrMIMd00/index.html


1. Ethernet Cloud connection 

Easy to setup in GNS3, but it bypasses the TCP/IP stack of the host machine. This means that somehting like a router would be needed on local LAN to allow the cloud guest and host to interact (because the router would route the packets to the linux host and vice versa).

First of all I changed the netplan configuration inside the cloud guest. The config is given as below screenshot.
![](https://i.imgur.com/HFCoxG9.png)

I also changed the configuration for the internet connection of the cloud guest. Screenshot is below.
![](https://i.imgur.com/WWvyndh.png)

Then the ping from inside the cloud guest worked.
![](https://i.imgur.com/huW3YuO.png)


This is the resulting topology below.
![](https://i.imgur.com/weuAgBR.png)


2. TAP Cloud connection
This works by creating a fake device at the Ethernet level (layer 2). This requiers extra setup because the external devices will not be able to discover the cloud guest on their own, and the host will have to act as a router or NAT.

I created the TAP connection with NetworkManager on hte host by using this guide: https://mail.gnome.org/archives/networkmanager-list/2016-January/msg00049.html



3. Linux Bridge and NAT Cloud connection
This appears to be similar to how docker operates [https://www.securitynik.com/2016/12/docker-networking-internals-how-docker_16.html]. To allow communication between guest and host a virtual bridge is used. Nat is used to allow easy connection to the internet. This was easier to setup than the TAP Cloud Connection.

source: https://www.bernhard-ehlers.de/blog/2017/07/11/gns3-cloud-linux.html

