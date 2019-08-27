# FIA Lab 1 - PXE & Booting
#### Artem Abramov SNE19

May I suggest viewing this document in your browser at address: 
https://github.com/temach/innopolis_university_reports/blob/master/FIA-Lab-1-PXE-Booting.md
Unfortunatelly rendering the document to PDF breaks some long lines and crops images.

## Task 1 - PXE

### Create a virtual machine and setup a PXE service

I decided to use GNS3 to manage the virtual machines and to control the 
network. GNS3 was already installed in my machine as part fo doing the NRI 
networking lab, detailed instructions for installation and configuration of 
GNS3 can be found there: 
https://github.com/temach/innopolis_university_reports/blob/master/INR-Lab-1-Networking-Basics.md

I created a new virtual machine. Installed KaliLinux on it, connected it to the cloud appliance in GNS3 (this allows it to access the internet) and configured its networking. The cloud applience is configured to use virbr0 virtual network bridge. This bridge is created after installing libvirt. Libvirt also sets up DHCP and NAT access to the internet for devices connected to the bridge.

Screenshot showing the the virtual network topology is below.
![](https://i.imgur.com/shq6ZOM.png)

Screenshot of the KaliLinux installer is below.
![](https://i.imgur.com/z4QwSPF.png)

Screenshot of the GNS3 cloud appliance configuration is shown below.
![](https://i.imgur.com/dsanzDj.png)

For reference below is a screenshot showing the version of KaliLinux guest that was used for this lab.
![](https://i.imgur.com/8ORez1h.png)

With everything ready its now important to understand what is PXE and the details of how it works. In my opinion the best overview is presented in the document that describes PXE, source: http://www.pix.net/software/pxeboot/archive/pxespec.pdf

To setup the PXE service I plan to use two packages: dnsmasq for DHCP+TFTP and pxelinux for the PXE.

If its impossible to use dnsmasq, the DHCP and TFTP servers can be setup 
separately, source: https://debian-administration.org/article/478/Setting_up_a_server_for_PXE_network_booting.



Small note: at this moment I decided to setup the ssh server to the guest os, because working with it through VNC was not very practical. Through VNC I allowed root login via ssh with just a password, this is shown on the screenshot below.
![](https://i.imgur.com/hHYiUkf.png)

Screenshot below shows the ssh server on the KaliLinux guest working.
![](https://i.imgur.com/Fj9O6Ci.png)

After this the console access to the guest was working. The next step was to install the packages.


Howerver there was a problem with installing the packages!
Because I installed the system from a CD image, the apt sources were not configured properly. In other words the system failed to find packages, because it was only using the apt repository that came with the CD. 

Below is a screenshot showing broken /etc/apt/sources.list configuration.
![](https://i.imgur.com/oFq7WdD.png)


The next step was adding the default KaliLinux repository: 
```
deb http://http.kali.org/kali kali-rolling main non-free contrib
```
source: https://docs.kali.org/general-use/kali-linux-sources-list-repositories

After this it was necessary to update the package lists with `apt-get update` and then to finally install pxelinux and dnsmasq.

The next step was configuring dnsmasq.
The information on how to setup dnsmasq was mostly taken from two links:
1. Oracle documentation https://docs.oracle.com/cd/E37670_01/E41137/html/ol-dnsmasq-conf.html
2. Blog post (viewed from Internet Archive) https://web.archive.org/web/20170710081151/https://blogging.dragon.org.uk/howto-setup-a-pxe-server-with-dnsmasq/

Edit the dnsmasq config to look as on the screenshot below:
![](https://i.imgur.com/Vg1ulcY.png)

Create the TFTP directory to match what was specified in dnsmasq config. Then restart dnsmasq and check that it found TFTP directory and config is ok, as show in screenshot below.
![](https://i.imgur.com/IDPl2VU.png)

Because KaliLinux is easier to work with than Ubuntu, this lab will use booting to KaliLinux with PXE as an example. The next step is to actually get the system files that are used by the KaliLinux distribution. 

For this step the instructions from https://kali.training/topic/installing-kali-over-the-network/ were very useful.

I discovered that KaliLinux provides official PXE boot files. Which include the pxelinux.0 and ldlinux.c32 from the pxelinux and syslinux-common packages. This greatly simplified the process.

So download the tarball for a graphical installer of KaliLinux on amd64 from 
http://http.kali.org/dists/kali-rolling/main/installer-amd64/current/images/netboot/gtk/netboot.tar.gz. And just unpack it /var/lib/tftpboot.

```
# cd /var/lib/tftpboot
# wget http://http.kali.org/dists/kali-rolling/main/installer-amd64/current/images/netboot/gtk/netboot.tar.gz
# tar xf netboot.tar.gz
```

Below is a screenshot showing the resulting tftpboot dir.
![](https://i.imgur.com/9aAFzHt.png)

There is the option to tweak boot files in `debian-installer/amd64/boot-screens/`, if you want to configure the installer.


### Question: why not run your DHCP service on the SNE network directly?

We do not know the topology of the network, is there is at least one more DHCP server running then the connected machines might get inconsistent ip addresses. This will cause connectivity issues. Two machines that are physically connected will be logically on two different networks. Alternatively its also possible that two machines will be given the same ip address, which will also create connection problems.

### Create yet another virtual machine to proceed with acceptance testing of your installation.

Create another VM in GNS3. The config of the VM is shown in the screenshot below. Note that the boot priority is set to Network.
![](https://i.imgur.com/vcALdRj.png)

I also removed the CD iso image from the VM config as shown on the screenshot below.
![](https://i.imgur.com/0sFHxxn.png)

The last step was disconnecting the DHCP that was provided by libvirt on virbr0. Now the KaliLinux box should provide DHCP service on the network. In fact the network was also isolated from the internet.

The resulting network topology is shown on the screenshot below.
![](https://i.imgur.com/UYgHL93.png)

On the first boot the dnsmasq server was not running, so the "no-os" machine failed with the error below.
![](https://i.imgur.com/0X7aSp4.png)

Then I realised that dnsmasq might be conflicting with NetworkManager on the PXE server. So I logged in and disabled NetworkManager service. The next part was configuring a static ip for the eth0 interface, so that the interface was up and ready for dnsmasq to use.

Below is a screenshot of the eth0 interface config on KaliLinux PXE server.
![](https://i.imgur.com/CRCE5dT.png)

After the PXE server booted I tried booting "no-os" machine again.
This time the boot was successfull and the welcoming screen was displayed!

Below is the screenshot of the welcoming screen.
![](https://i.imgur.com/xzqykHO.png)

Net installer uses a menu as show on the screenshot below.
![](https://i.imgur.com/chPQuNv.png)


## Task 2 - Questions

### 1. What is UEFI PXE booting? How does it work? How does it compare to booting from the hard disk or a CD?

UEFI PXE booting refers to booting over PXE by clients that use UEFI instead of Legacy BIOS. UEFI PXE differs from legacy PXE in that the client uses UEFI module to interact with the PXE server. If the client uses legacy BIOS then the code to interact with the PXE server would normally be provided as a Network Interface Card (NIC) BIOS extension. The PXE server does not care to which type of client the service is provided. However to handle both types of clients the server must provide a binary to execute on a legacy BIOS client (such as pxelinux) and an alternative binary (such as elilo) to execute on UEFI clients. The clients will themselves decide which binary they need.

PXE booting works as show in the diagram below:
![](https://i.imgur.com/QRwhow4.png)

The booting machine tries to discover services using the DHCP protocol. The server responds with a DHCP packet that contains extra information about pxe booting. The client can use this information to access the TFTP server. 
After downloading the files from TFTP server and veryfying them, the client can execute them to start the boot or install process.

Compared to booting from a CD or HardDrive the PXE has several advantages. First of all it does not require any physical installation media, beyound thouse that are likely to be present in any computer anyway (network card).
Secondly it allows to remotely controll the installion of an operating system on a machine.

source: https://web.archive.org/web/20131102003141/http://download.intel.com/design/archives/wfm/downloads/pxespec.pdf



### 2. What is a GPT? What is its layout? What is the role of a partition table?

GPT is the modern alternative to MBR. A specification describing how to divide disk into partitions and to keep track of the partition table. GPT also removes some of the limitations of MBR, such as a much higher limit on partition size, limit of on the number of partitions. GPT also keeps a dublicate partition table on disk that can be used for recovery (unlike MBR).

Below is a diagram of an example disk that is using GPT.

![](https://i.imgur.com/t3xOXhB.png)

The layout can be described as follows:

1. First 512 bytes are left untouched to allow compatiability with legacy boot BIOS.
2. Then follows the header for the GPT table.
3. Then follow the entries in the GPT table. Each entry describes a partition.
4. After the end of the table, there is space to place the actual partitions.
5. Finally at the end of the storage medium, the GPT partition table is duplicated.

The partition table describes how the storage medium is divided into logical blocks and what special attributes does each block hold. This is of primary importance to tools that work at the operating system level. Such as the bootloader, the OS itself, etc. As the attributes attached to each partition describe what such low-level tools are allowed to do with the partition (for example to check if the partition is bootable, or what filesystem is uses). For userspace programs usually the operating system abstracts the storage medium by providing the filesystem abstraction. However there are many legitimate cases when the software wants to work directly with the data on the disk, flash drive or CD. For such cases the partitions describe how the program must handle the different areas of the storage medium.

### 3. What is gdisk? How does it work? What can you do with it?

gdisk is a linux utility to create, modify and delete partitions and partiton table on the GPT drives. It comes pre-installed on ubuntu. It is a successor to the fdisk program, that can only work with MBR disks. The program actually comes in three flavours of user interface.

1. gdisk is text-mode interactive
2. sgdisk is command-line
3. cgdisk has a curses-based interface

gdisk is a usrespace program that accesses the underlying storage medium to manipulate the GPT partitions. You can use to manipulate the GPT partition in any way that is allowed by the UEFI specification. In particular it allows the creation of GPT disks with protectuve MBR section.

source: https://www.rodsbooks.com/gdisk/walkthrough.html

### 4. What is a Protective MBR and why is it in the GPT?

As already outlined in the second question above the main usage for protective MBR is to allow compatiability with legacy systems and programs that do not support working with GPT partitions. This concerns USB drives that would not be functional on systems without GPT support, perhaps even more importantly this concerns the hard drives that would become unbootable when plugged into a motherboard with a BIOS that does not understand GPT partitions. 
Itself the protective MBR is just the normal MBR partition table that makes the best effort to describe the storage medium. If the disk has for example 10 partitions that are in the GPT table, only the first 4 partitions will recorded in the MBR table.

## Task 3 - Partitions
