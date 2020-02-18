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
apt install git autoconf automake gcc libtool make file pkg-config libjson-c-dev uuid-dev libtirpc-dev targetcli-fb cmake libnl-3-dev libnl-genl-3-dev libglib2.0-dev zlib1g-dev kmod libkmod-dev libgoogle-perftools-dev
```

Clone sources and build:

```
git clone https://github.com/gluster/gluster-block
cd gluster-block
./autogen.sh
./configure --with-systemddir=/usr/lib/systemd/system --enable-tirpc=no
```

Make sure that directory from the config actually exists (not by default):

```
mkdir -p /usr/lib/systemd/system
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



`targetcli-fb` is another runtime dependency, but we installed it earlier.

Enable glsuter-blockd and tcmu-runner systemd files:

```
cd /usr/lib/systemd/system
systemctl enable tcmu-runner.service 
systemctl start tcmu-runner.service 
systemctl enable gluster-block-target.service gluster-blockd.service 
```



However, this is far from the end!

There are multiple errors to overcome. First fix dependency versions.

Remove Ubuntu packages.

```
apt-get remove targetcli-fb python-rtslib-fb python3-configshell-fb python3-rtslib-fb
```

Install pip:

```
apt-get install python-pip
```

Install the packages as below:

```
asn1crypto (0.24.0)
configshell-fb (1.1.25)
cryptography (2.1.4)
enum34 (1.1.6)
idna (2.6)
ipaddress (1.0.17)
keyring (10.6.0)
keyrings.alt (3.0)
pip (9.0.1)
pycrypto (2.6.1)
pygobject (3.26.1)
pyparsing (2.4.6)
pyudev (0.22.0)
pyxdg (0.25)
rtslib-fb (2.1.71)
SecretStorage (2.3.1)
setuptools (39.0.1)
six (1.11.0)
targetcli-fb (2.1.51)
urwid (2.0.1)
wheel (0.30.0)
```

Currently the latest `uwrid(2.1.x)` has problems, so use `urwid(2.0.1)` .

Pay particular attention to `rtslib-fb`, `targetcli-fb` , `configshell-fb` packages. 



For the `rtslib-fb`  get the source to install `systemd` file:

```
git clone https://github.com/open-iscsi/rtslib-fb
cd rtslib-fb/
cp systemd/target.service /usr/lib/systemd/system/
```



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

Change to the directory and enable all services:

```
cd /usr/lib/systemd/system
systemctl daemon-reload
systemctl enable *
systemctl start gluster-blockd.service
```





If things break test in particular that `targetcli` command works as shown here:

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





Finally we can create block device on our hosting volume!

```
gluster-block create sample/block 10.1.1.97 1GiB --json-pretty
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
gluster-block info sample/block --json-pretty
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
# gluster peer status
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

##### Configure the client

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



Create data inside:

```
cd /mnt/gfs-sample/
sudo mkdir artem
sudo touch artem/a.txt artem/b.txt artem/c.txt
```

Then unmount:

```
sudo umount /mnt/gfs-sample/
```



To complete install client tools on another machine, mount and verify that files are present, and indeed they are.



