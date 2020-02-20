# LS Lab 4 - Distributed SAN

Artem Abramov

### Individual: run your storage farm locally, in guests

### Take a pick

I choose GlusterFS

sources:

- https://github.com/gluster/gluster-block

- https://github.com/gluster/gluster-block/issues/215

- https://kifarunix.com/install-and-setup-glusterfs-on-ubuntu-18-04/

- https://www.scaleway.com/en/docs/how-to-configure-storage-with-glusterfs-on-ubuntu/

- https://www.cyberciti.biz/faq/howto-glusterfs-replicated-high-availability-storage-volume-on-ubuntu-linux/

  

I prepared three virtual xen machines.

For each machine I created an new drive, just to hold the glusterfs.

```
dd if=/dev/zero of=guest1.img bs=1024k seek=6144 count=1
```



On each machine (and on dom0) I created the same /etc/hosts file to make naming easier:

```
10.1.1.97		labguest1.local
10.1.1.184 		labguest3.local
10.1.1.77       labguest4.local
```

Note: a common issue is `Temoporary name resolution error` and to avoid it make sure to use the full hostnames as you defined them in /etc/hosts in all later commands.

Add Gluster repository (I used the version 5, but versions 6 and even 7 are available)

```
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:gluster/glusterfs-5
```

Update

```
apt update
```

Install server and enable

```
apt install glusterfs-server
systemctl enable glusterd
systemctl daemon-reload
systemctl start glusterd
```



##### Here we can either create a pool first or create a volume first. 

I decided to create the volume first (on one server node), test that is works, and then add peer server nodes to the pool.

Create volume to host the block device:

```
# gluster vol create sample labguest1.local:/brick force
volume create: sample: success: please start the volume to access data
```

Start the volume:

```
# gluster vol start sample
volume start: sample: success
```



Check volume info:

```
# gluster volume info
 
Volume Name: sample
Type: Distribute
Volume ID: 878c9044-8b80-495c-9ff7-e69ffcb95efc
Status: Started
Snapshot Count: 0
Number of Bricks: 1
Transport-type: tcp
Bricks:
Brick1: labguest1.local:/brick
Options Reconfigured:
transport.address-family: inet
nfs.disable: on
```

Check volume info again:

```
# gluster volume status
Status of volume: sample
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick labguest1.local:/brick                 49152     0          Y       23679
 
Task Status of Volume sample
------------------------------------------------------------------------------
There are no active volume tasks
```



Now on top of this block hosting volume, we must create a glusterfs-block device (do this on one of the server nodes). For this use the repository: https://github.com/gluster/gluster-block

Install build-time dependencies:

```
apt install git autoconf automake gcc libtool make file pkg-config libjson-c-dev uuid-dev libtirpc-dev cmake libnl-3-dev libnl-genl-3-dev libglib2.0-dev zlib1g-dev kmod libkmod-dev libgoogle-perftools-dev
```

targetcli-fb

Clone sources and build (I was building from commit 9767ecea8493276d217044a195b640d4ff85e28d made on 2020-01-03):

```
git clone https://github.com/gluster/gluster-block
cd gluster-block
```

Decide where to put systemd files to control the daemons, I will use `/usr/lib/systemd/system`. 

**IMPORTANT** before running autogen and configure ensure that directory from the config actually exists (it does not by default):

```
mkdir -p /usr/lib/systemd/system
```

Make sure directory exists and then run configure:

```
./autogen.sh
./configure --with-systemddir=/usr/lib/systemd/system --enable-tirpc=no
```

Configuration summary:

```
------------------ Summary ------------------
 gluster-block version 0.4
  Prefix.........: /usr/local
  C Compiler.....: gcc -g -O2 
  Linker.........: /usr/bin/ld -m elf_x86_64  
  Using Tirpc....: no
---------------------------------------------
```

Build and install:

```
make -j install
```

**IMPORTANT** Modify the file `/usr/lib/systemd/system/gluster-block-target.service` to include the line (this is taken by following the error in `journalctl -xe` ):

```
[Unit]
RemainAfterExit=yes
```





Download and install runtime dependencies: Tcmu-runner

```
git clone https://github.com/open-iscsi/tcmu-runner
cd tcmu-runner
cmake -DSUPPORT_SYSTEMD=ON -DCMAKE_INSTALL_PREFIX=/usr -Dwith-rbd=false -Dwith-qcow=false -Dwith-zbc=false -Dwith-fbo=false
make install
```



At this point we should really be able to run gluster-block. However, in reality this is far from the end!

There are multiple errors to overcome. First fix dependency versions, by removing outdated Ubuntu packages (they should be absent by default) and installing newer versions from pip:

```
apt-get remove targetcli-fb python-rtslib-fb python3-rtslib-fb python3-configshell-fb python3-urwid python3-pyparsing 
```

Install pip:

```
apt-get install python-pip
```

Versions of default pip packages:

```
asn1crypto==0.24.0
cryptography==2.1.4
enum34==1.1.6
idna==2.6
ipaddress==1.0.17
keyring==10.6.0
keyrings.alt==3.0
pip==9.0.1
pycrypto==2.6.1
pygobject==3.26.1
pyxdg==0.25
SecretStorage==2.3.1
six==1.11.0
wheel==0.30.0
```

Now to install gluster-block dependencies from file with pip use:

```
pip install -r glusterdeps.txt
```

Install the packages with versions as below:

```
# cat glusterdeps.txt
configshell-fb==1.1.25
pyparsing==2.4.6
pyudev==0.22.0
rtslib-fb==2.1.71
targetcli-fb==2.1.51
urwid==2.0.1
```

Currently the latest `uwrid(2.1.x)` has problems, so use `urwid(2.0.1)` .

Pay particular attention to `rtslib-fb`, `targetcli-fb` , `configshell-fb` packages, they might be out of sync and if things break suspect them first. Uninstalling the pip versions and reinstalling from source helps in this case.06076aba7e9e9bd4a1e84bac61e85265e8075b8e



##### Slight detour to create targetcli-fb pip package

I was faced with the problem that `targetcli-fb` was not available in the Python Package Index. I decided to do a service to everyone by packaging and uploading it there (otherwise I would have to install it on all three server nodes from source).

To package code for pip, first register on the https://pypi.org/

```
username: temach1
email: aabramovrussia@gmail.com
```

Clone the code:

```
git clone https://github.com/open-iscsi/targetcli-fb
```

Install dependencies (setuptool to build package, twine to upload it to PyPi):

```
apt install python-setuptools twine
```

Create package and upload:

```
cd targetcli-fb/
python setup.py sdist
twine upload dist/*
```

##### End of slight detour



The preferred install order is:

- targetcli-fb
- rtslib-fb
- configshell-fb



For `targetcli-fb` get the source to install the `systemd` file:

```
git clone https://github.com/open-iscsi/targetcli-fb
cd targetcli-fb
cp systemd/targetclid.socket /usr/lib/systemd/system/
cp systemd/targetclid.service /usr/lib/systemd/system/
```

For  `targetcli-fb` make symbolic links (because python packages install into `/usr/local/bin` but others expect them at `/usr/bin`, this is seen from `journalctl -xe` errors): 

```
ln -s /usr/local/bin/targetclid /usr/bin/targetclid
ln -s /usr/local/bin/targetctl /usr/bin/targetctl
ln -s /usr/local/bin/targetcli /usr/bin/targetcli
```



For the `rtslib-fb`  get the source to install `systemd` file:

```
git clone https://github.com/open-iscsi/rtslib-fb
cd rtslib-fb/
cp systemd/target.service /usr/lib/systemd/system/
```



Next try to run  `targetcli` command as shown here:

https://yari.net/2016/08/28/how-to-fix-not-working-targetclitarget-on-centos-7-1-1503/

```
# targetcli 
targetcli shell version 2.1.51
Entering targetcli batch mode for daemonized approach.
Enter multiple commands separated by newline and type 'exit' to run them all in one go.

/> ls
/> exit
o- / ......................................................... [...]
  o- backstores .............................................. [...]
  | o- block .................................. [Storage Objects: 0]
  | o- fileio ................................. [Storage Objects: 0]
  | o- pscsi .................................. [Storage Objects: 0]
  | o- ramdisk ................................ [Storage Objects: 0]
  | o- user:glfs .............................. [Storage Objects: 0]
  o- iscsi ............................................ [Targets: 0]
  o- loopback ......................................... [Targets: 0]
  o- vhost ............................................ [Targets: 0]
  o- xen-pvscsi ....................................... [Targets: 0]
```

Most likely you will need to create the config directory that is will populate with default settings:

```
mkdir -p /etc/target
```





Final configuration for directory `/usr/lib/systemd/system`:

```
root@labguest1:/usr/lib/systemd/system# ls -la
total 32
drwxr-xr-x  2 root root 4096 Feb 18 00:55 .
drwxr-xr-x 12 root root 4096 Feb 17 23:53 ..
-rw-r--r--  1 root root  552 Feb 18 00:49 gluster-block-target.service
-rw-r--r--  1 root root  543 Feb 18 00:09 gluster-blockd.service
-rw-r--r--  1 root root  331 Feb 18 00:55 target.service
-rw-r--r--  1 root root  222 Feb 18 00:35 targetclid.service
-rw-r--r--  1 root root  152 Feb 18 00:35 targetclid.socket
-rw-r--r--  1 root root  303 Feb 17 23:43 tcmu-runner.service
```

Change to the directory and just to be sure first stop all, then enable all, and start all services:

```
cd /usr/lib/systemd/system
systemctl stop *
systemctl daemon-reload
systemctl enable *
systemctl start gluster-blockd.service
```





If things break test in particular that `targetcli` command works:

https://yari.net/2016/08/28/how-to-fix-not-working-targetclitarget-on-centos-7-1-1503/

```
# targetcli 
targetcli shell version 2.1.51
Entering targetcli batch mode for daemonized approach.
Enter multiple commands separated by newline and type 'exit' to run them all in one go.

/> ls
/> exit
o- / ......................................................... [...]
  o- backstores .............................................. [...]
  | o- block .................................. [Storage Objects: 0]
  | o- fileio ................................. [Storage Objects: 0]
  | o- pscsi .................................. [Storage Objects: 0]
  | o- ramdisk ................................ [Storage Objects: 0]
  | o- user:glfs .............................. [Storage Objects: 0]
  o- iscsi ............................................ [Targets: 0]
  o- loopback ......................................... [Targets: 0]
  o- vhost ............................................ [Targets: 0]
  o- xen-pvscsi ....................................... [Targets: 0]
```

Its also possible that targetcli fails and complains about daemon needed. In this particular error just go directly for starting the gluster-block daemon:

```
systemctl start gluster-blockd.service
```

Here is the output of a healthy gluster-blockd.service:

```
● gluster-blockd.service - Gluster block storage utility
   Loaded: loaded (/usr/lib/systemd/system/gluster-blockd.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-02-20 01:13:20 UTC; 3min 42s ago
 Main PID: 22987 (gluster-blockd)
    Tasks: 3 (limit: 1117)
   CGroup: /system.slice/gluster-blockd.service
           ├─22987 /usr/local/sbin/gluster-blockd --glfs-lru-count 5 --log-level INFO
           └─23081 /usr/local/sbin/gluster-blockd --glfs-lru-count 5 --log-level INFO

Feb 20 01:13:20 labguest4 systemd[1]: Started Gluster block storage utility.
Feb 20 01:13:21 labguest4 gluster-blockd[22987]: Parameter logfile is now '/var/log/gluster-block/gluster-block-configshell.log'.
Feb 20 01:13:21 labguest4 gluster-blockd[22987]: Parameter loglevel_file is now 'info'.
Feb 20 01:13:21 labguest4 gluster-blockd[22987]: Parameter auto_use_daemon is now 'true'.
Feb 20 01:13:21 labguest4 gluster-blockd[22987]: Parameter auto_enable_tpgt is now
Feb 20 01:13:21 labguest4 gluster-blockd[22987]: Parameter auto_add_default_portal is now
Feb 20 01:13:21 labguest4 gluster-blockd[22987]: Parameter auto_save_on_exit is now 
Feb 20 01:13:21 labguest4 gluster-blockd[22987]: Parameter max_backup_files is now '100'.
```





Finally we can create block device on our hosting volume!

```
# gluster-block create sample/block 10.1.1.97 1GiB --json-pretty
{
  "IQN":"iqn.2016-12.org.gluster-block:ee4f68fe-890d-426b-8cc7-6fd787761a7f",
  "PORTAL(S)":[
    "10.1.1.97:3260"
  ],
  "RESULT":"SUCCESS"
}
```

And we can delete it with:

```
gluster-block delete sample/block --json-pretty
```

Check with:

```
# gluster-block info sample/block --json-pretty
{
  "NAME":"block",
  "VOLUME":"sample",
  "GBID":"ee4f68fe-890d-426b-8cc7-6fd787761a7f",
  "SIZE":"1.0 GiB",
  "HA":1,
  "IOTIMEOUT":"43 Seconds",
  "PASSWORD":"",
  "EXPORTED ON":[
    "10.1.1.97"
  ]
}
```





##### For failed `gluster` CLI commands, see default log with:

```
less /var/log/glusterfs/glusterd.log 
```





##### Add the other two nodes to the server pool. 

Useful source: https://www.cyberciti.biz/faq/howto-add-new-brick-to-existing-glusterfs-replicated-volume/

Setup the new node according to the instructions in the beginning, until the point of creating a gluster volume to host block devices. Then configure the pool by probing peer server node from the existing node:

```
root@labguest1:~# gluster peer probe labguest3.local
peer probe: success.
```

Check status and list:

```
gluster peer status
gluster pool list
```



Add the brick to the volume, better run on the host that created the volume (specify either `replica N+1` or `stripe N+1`):

```
gluster volume add-brick sample replica 2 labguest3.local:/brick force
```

To remove use (similarly when removing bricks specify `N-1`) and better run on the host that created the volume:

```
gluster volume remove-brick sample replica 1 labguest3.local:/brick force
```



Add two server nodes to the pool.

Check peer status (this was run on labguest4.local machine):

```
e gluster-block CLI from any of the 3 nodes where glusterd and gluster-blockd are running.# gluster peer status
Number of Peers: 2

Hostname: labguest3.local
Uuid: 2b6faeeb-a75e-4551-a03c-67e58ff8162e
State: Peer in Cluster (Connected)

Hostname: labguest1.local
Uuid: fbadc544-245d-43d0-82c6-c46e4cc5a52f
State: Peer in Cluster (Connected)
```

Check pool list (this was run on labguest4.local machine):

```
# gluster pool list
UUID					Hostname       	State
2b6faeeb-a75e-4551-a03c-67e58ff8162e	labguest3.local	Connected 
fbadc544-245d-43d0-82c6-c46e4cc5a52f	labguest1.local	Connected 
449e3208-3139-42b9-8b1e-6e1d063a2a77	localhost      	Connected
```



Add two server nodes as replicas to the block device hosting volume.

Check volume info:

```
# gluster vol info
 
Volume Name: sample
Type: Replicate
Volume ID: a54e19c9-eba2-4d76-804c-5c64d728e400
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp
Bricks:
Brick1: labguest1.local:/brick
Brick2: labguest3.local:/brick
Brick3: labguest4.local:/brick
Options Reconfigured:
nfs.disable: on
transport.address-family: inet
performance.client-io-threads: off
```

Check volume status:
```
# gluster vol status
Status of volume: sample
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick labguest1.local:/brick                49152     0          Y       7366 
Brick labguest3.local:/brick                49152     0          Y       1948 
Brick labguest4.local:/brick                49152     0          Y       6860 
Self-heal Daemon on localhost               N/A       N/A        Y       6883 
Self-heal Daemon on labguest3.local         N/A       N/A        Y       2253 
Self-heal Daemon on labguest1.local         N/A       N/A        Y       7457 
 
Task Status of Volume sample
------------------------------------------------------------------------------
There are no active volume tasks
```



Its also possible to create volumes and tie them to bricks immediately (3 replicas is a decent setup):

```
# gluster volume create <volname> replica 3 host1:brick1 host2:brick2 host3:brick3
```



### Acceptance Testing

First step is to check that GlusterFS itself works, without the block device. After that I will provide acceptance testing of the block-device. 

##### Testing GlusterFS

source: https://www.scaleway.com/en/docs/how-to-configure-storage-with-glusterfs-on-ubuntu/

I used my host machine to test as a client (dom0):

```
apt install glusterfs-client
```

Create mount dir and mount:

```
mkdir -p /mnt/gfs-sample
sudo mount -t glusterfs labguest1.local:/sample /mnt/gfs-sample
```



Check the mount type:

```
# df -hTP /mnt/gfs-sample/
Filesystem              Type            Size  Used Avail Use% Mounted on
labguest1.local:/sample fuse.glusterfs  5,9G  4,4G  1,3G  79% /mnt/gfs-sample
```

Note: the Avail space is just 1.3G, this is because the cluster is set to replicate data between three bricks and one of the bricks has only 1.3G left, so with current configuration the space available in the cluster is the space of the smallest brick.

Here is the smallest brick:

```
root@labguest1:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
udev            466M     0  466M   0% /dev
tmpfs            98M  400K   98M   1% /run
/dev/xvda1      5.9G  4.4G  1.3G  78% /
tmpfs           489M   12K  489M   1% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           489M     0  489M   0% /sys/fs/cgroup
tmpfs            98M     0   98M   0% /run/user/0
```



Looking inside the glusterfs there is data placed there by the gluster-block addon (in particular the 1 gigabyte file for the block device is there):

```
artem@labubuntu:/mnt/gfs-sample$ sudo ls -Rlh /mnt/gfs-sample
.:
total 8,0K
d--------- 2 root root 4,0K фев 18 06:02 block-meta
d--------- 2 root root 4,0K фев 18 05:56 block-store

./block-meta:
total 512
-rw------- 1 root root 238 фев 18 04:48 block
-rw------- 1 root root   0 фев 18 04:47 meta.lock
-rw------- 1 root root   0 фев 18 04:47 prio.info

./block-store:
total 1,0G
-rw------- 1 root root 1,0G фев 18 04:48 ee4f68fe-890d-426b-8cc7-6fd787761a7f
```



Create data inside to test GlusterFS:

```
cd /mnt/gfs-sample/
sudo mkdir artem
sudo touch artem/a.txt artem/b.txt artem/c.txt
```

Then unmount:

```
sudo umount /mnt/gfs-sample/
```

To complete install client tools on another machine (I used labguest4.local), mount and verify that files are present, and indeed they are.

This concludes testing GlusterFS.



##### Testing gluster-block

Now its time to setup the other 2 server nodes with gluster-blockd. Get the gluster-blockd.service running on all the 3 server nodes. Check availability from other nodes with the above gluster-block info command.

Then create a new block device that is spread across two server nodes:

```
# gluster-block create sample/block2 ha 2 10.1.1.97,10.1.1.184 1GiB --json-pretty
{
  "IQN":"iqn.2016-12.org.gluster-block:38b6b1aa-0332-4fde-b0ab-e27be04ec65d",
  "PORTAL(S)":[
    "10.1.1.97:3260",
    "10.1.1.184:3260"
  ],
  "RESULT":"SUCCESS"
}
```



Install iSCSI (Internet Small Computer System Interface) initiator:

```
sudo apt install open-iscsi
```

Search for block devices:

```
sudo iscsiadm -m discovery -t st -p labguest3.local
10.1.1.97:3260,1 iqn.2016-12.org.gluster-block:38b6b1aa-0332-4fde-b0ab-e27be04ec65d
10.1.1.184:3260,2 iqn.2016-12.org.gluster-block:38b6b1aa-0332-4fde-b0ab-e27be04ec65d
```

So there is one device found!

List current block devices:

```
# lsblk
NAME   MAJ:MIN RM    SIZE RO TYPE MOUNTPOINT
loop0    7:0    0   1008K  1 loop /snap/gnome-logs/61
loop1    7:1    0   54,7M  1 loop /snap/core18/1668
loop2    7:2    0   54,4M  1 loop /snap/core18/1066
loop3    7:3    0   89,1M  1 loop /snap/core/8268
loop4    7:4    0   42,8M  1 loop /snap/gtk-common-themes/1313
loop5    7:5    0  160,2M  1 loop /snap/gnome-3-28-1804/116
loop6    7:6    0    3,7M  1 loop /snap/gnome-system-monitor/127
loop7    7:7    0   14,8M  1 loop /snap/gnome-characters/399
loop8    7:8    0    3,7M  1 loop /snap/gnome-system-monitor/100
loop9    7:9    0      6G  0 loop 
loop10   7:10   0      6G  0 loop 
loop11   7:11   0   14,8M  1 loop /snap/gnome-characters/296
loop12   7:12   0      4M  1 loop /snap/gnome-calculator/406
loop13   7:13   0  131,9M  1 loop /snap/telegram-desktop/1038
loop14   7:14   0    4,2M  1 loop /snap/gnome-calculator/544
loop15   7:15   0  149,9M  1 loop /snap/gnome-3-28-1804/67
loop16   7:16   0   44,9M  1 loop /snap/gtk-common-themes/1440
loop17   7:17   0    956K  1 loop /snap/gnome-logs/81
loop18   7:18   0      1G  0 loop 
loop19   7:19   0   91,3M  1 loop /snap/core/8592
loop20   7:20   0      6G  0 loop 
loop21   7:21   0      1G  0 loop 
loop22   7:22   0      1G  0 loop 
sda      8:0    0  931,5G  0 disk 
├─sda1   8:1    0    512M  0 part /boot/efi
├─sda2   8:2    0  541,3G  0 part 
├─sda3   8:3    0 1023,7M  0 part [SWAP]
└─sda4   8:4    0  388,8G  0 part /
sr0     11:0    1   1024M  0 rom 
```



Now scan and login (note the trailing `-l`):

```
# sudo iscsiadm -m discovery -t st -p labguest3.local -l
10.1.1.97:3260,1 iqn.2016-12.org.gluster-block:38b6b1aa-0332-4fde-b0ab-e27be04ec65d
10.1.1.184:3260,2 iqn.2016-12.org.gluster-block:38b6b1aa-0332-4fde-b0ab-e27be04ec65d
Logging in to [iface: default, target: iqn.2016-12.org.gluster-block:38b6b1aa-0332-4fde-b0ab-e27be04ec65d, portal: 10.1.1.97,3260] (multiple)
Logging in to [iface: default, target: iqn.2016-12.org.gluster-block:38b6b1aa-0332-4fde-b0ab-e27be04ec65d, portal: 10.1.1.184,3260] (multiple)
Login to [iface: default, target: iqn.2016-12.org.gluster-block:38b6b1aa-0332-4fde-b0ab-e27be04ec65d, portal: 10.1.1.97,3260] successful.
Login to [iface: default, target: iqn.2016-12.org.gluster-block:38b6b1aa-0332-4fde-b0ab-e27be04ec65d, portal: 10.1.1.184,3260] successful.
```

List current block devices again:

```
# lsblk
NAME   MAJ:MIN RM    SIZE RO TYPE MOUNTPOINT
loop0    7:0    0   1008K  1 loop /snap/gnome-logs/61
loop1    7:1    0   54,7M  1 loop /snap/core18/1668
loop2    7:2    0   54,4M  1 loop /snap/core18/1066
loop3    7:3    0   89,1M  1 loop /snap/core/8268
loop4    7:4    0   42,8M  1 loop /snap/gtk-common-themes/1313
loop5    7:5    0  160,2M  1 loop /snap/gnome-3-28-1804/116
loop6    7:6    0    3,7M  1 loop /snap/gnome-system-monitor/127
loop7    7:7    0   14,8M  1 loop /snap/gnome-characters/399
loop8    7:8    0    3,7M  1 loop /snap/gnome-system-monitor/100
loop9    7:9    0      6G  0 loop 
loop10   7:10   0      6G  0 loop 
loop11   7:11   0   14,8M  1 loop /snap/gnome-characters/296
loop12   7:12   0      4M  1 loop /snap/gnome-calculator/406
loop13   7:13   0  131,9M  1 loop /snap/telegram-desktop/1038
loop14   7:14   0    4,2M  1 loop /snap/gnome-calculator/544
loop15   7:15   0  149,9M  1 loop /snap/gnome-3-28-1804/67
loop16   7:16   0   44,9M  1 loop /snap/gtk-common-themes/1440
loop17   7:17   0    956K  1 loop /snap/gnome-logs/81
loop18   7:18   0      1G  0 loop 
loop19   7:19   0   91,3M  1 loop /snap/core/8592
loop20   7:20   0      6G  0 loop 
loop21   7:21   0      1G  0 loop 
loop22   7:22   0      1G  0 loop 
sda      8:0    0  931,5G  0 disk 
├─sda1   8:1    0    512M  0 part /boot/efi
├─sda2   8:2    0  541,3G  0 part 
├─sda3   8:3    0 1023,7M  0 part [SWAP]
└─sda4   8:4    0  388,8G  0 part /
sdc      8:32   0      1G  0 disk 
sdd      8:48   0      1G  0 disk 
sr0     11:0    1   1024M  0 rom  sudo iscsiadm -m discovery -t st -p labguest3.local
```



We can see that there is `/dev/sdc` and `/dev/sdd` blocks and they are 1G in size!

 There are actually two block devices that lead to the same object, however for proper sync the client should install and use `multipath` tools.

Make the file system and mount it:

```
# sudo mkfs.ext4 /dev/sdc
mke2fs 1.44.1 (24-Mar-2018)
Discarding device blocks: done                            
Creating filesystem with 262144 4k blocks and 65536 inodes
Filesystem UUID: f2129a52-adec-4e20-874f-28faa20903df
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done
```

Finally mount it:

```
mount /dev/sdc /mnt/gluster
```

Create a file. 

Then unmount. Then logout:

```
# sudo iscsiadm -m node --logout all
Logging out of session [sid: 2, target: iqn.2016-12.org.gluster-block:38b6b1aa-0332-4fde-b0ab-e27be04ec65d, portal: 10.1.1.97,3260]
Logging out of session [sid: 3, target: iqn.2016-12.org.gluster-block:38b6b1aa-0332-4fde-b0ab-e27be04ec65d, portal: 10.1.1.184,3260]
Logout of [sid: 2, target: iqn.2016-12.org.gluster-block:38b6b1aa-0332-4fde-b0ab-e27be04ec65d, portal: 10.1.1.97,3260] successful.
Logout of [sid: 3, target: iqn.2016-12.org.gluster-block:38b6b1aa-0332-4fde-b0ab-e27be04ec65d, portal: 10.1.1.184,3260] successful.
```

Verify that command below is empty (all sessions are closed):

```
sudo iscsiadm -m session
```

Check that its gone in `lsblk`.

Then change to another machine and mount the directory, the file is indeed there.