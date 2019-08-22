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

### 2. Starting a GNS3 Project

Setup GNS3 window is below.
![](https://i.imgur.com/Q9nUeJS.png)

Choosing the Ubuntu Cloud Guest project from template is below.
![](https://i.imgur.com/WcctZE2.png)

Giving the project a name is below.
![](https://i.imgur.com/hkrXkch.png)

