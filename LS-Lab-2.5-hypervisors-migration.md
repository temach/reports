# LIA Lab 2 - Clouds & Hypervisors



### Now as a team, choose which one of the two machines is also going to provide the shared storage for virtual disks to live in. Set it up and share both ideally, guests virtual disks and configurations.

- Take team member 1’s favorite guest (virtual disk and configuration) and put it in the -shared storage
- Take team member 2’s favorite guest (virtual disk and configuration) and put it in the shared storage
- Eventually fix the pathes in the configuration and validate that both guests run as well as before

I setup a guest according to the article: 
https://www.virtuatopia.com/index.php/Building_a_Debian_or_Ubuntu_Xen_Guest_Root_Filesystem_using_debootstrap

My initial guest config was as follows (also taken almost verbatim):
```
name="guest1"
vcpus=1
memory=1024
builder="pv"
disk = ['tap:aio:/home/artem/UbuntuXen.img,xvda1,w', 'tap:aio:/home/artem/UbuntuXen.swap,xvda2,w']
kernel="/boot/vmlinuz-5.3.0-28-generic"
ramdisk="/boot/initrd.img-5.3.0-28-generic"
root="/dev/xvda1 rw"
vif = [ '' ]
extra='xencons=tty'
```

I had an idea of booting with `pygrub` instead of using DirectBoot xen feature (via `kernel` and `ramdisk` params, but I was not able to configure pygrub properly).



Then I accesses the NFS server setup by Gaspar at 10.1.1.163:

```
mount -t 10.1.1.163:/private /mnt/private
```



I copied the files necessary for my VM to there, renaming them in the process, creating the following files:

```
ls -la /mnt/private/artem/
total 1048312
drwxr-xr-x 2 artem artem       4096 фев  8 22:06 .
drwxrwxrwx 3 root  artem       4096 фев  8 20:33 ..
-rwxrwxrwx 1 artem artem        306 фев  8 22:06 guest1.cfg
-rwxrwxrwx 1 artem artem 6443499520 фев  8 22:26 guest1.img
-rwxrwxrwx 1 artem artem 1073741824 фев  8 20:36 guest1.swap
-rwxrwxrwx 1 artem artem   39276413 фев  8 21:01 initrd.img-5.3.0-28-generic
-rwxr-xr-x 1 artem artem    9146616 фев  8 21:04 vmlinuz-5.3.0-28-generic
```



I fixed my config (basically the paths) to be as below:

```
name="guest1"
vcpus=1
memory=1024
builder="pv"
disk = ['tap:aio:/mnt/private/artem/guest1.img,xvda1,w', 'tap:aio:/mnt/private/artem/guest1.swap,xvda2,w']
kernel="/mnt/private/vmlinuz-5.3.0-28-generic"
ramdisk="/mnt/private/initrd.img-5.3.0-28-generic"
root="/dev/xvda1 rw"
vif = [ '' ]
extra='xencons=tty'
```

I verifies that I could run a guest from that image with `sudo xl create guest1.cfg -c` in that directory.



#### Now shut them down and run them on the other team member’s machine/host (cold-migration)

Prior to migration Gaspar had to permit root login in his SSH server config. Furthermore by default the root account on ubuntu is locked, to unlock it follow: https://askubuntu.com/questions/44418/how-to-enable-root-login

Initially I went straight for live migration, and managed to successfully migrate from my machine to Gaspar's machine, therefore to show the cold migration I will be migrating my guest back from Gaspar's machine to my machine. So I had to unlock root password, start the ssh server and permit root login.

Then I ssh as root to Gaspar's machine:

```
ssh root@10.1.1.143
```

 

Pause the domain:

```
root@baye:~# sudo xl pause guest1
root@baye:~# xl list
Name                                        ID   Mem VCPUs	State	Time(s)
Domain-0                                     0 13984     4     r-----   56475.2
guest1                                      28  1024     1     --p---      28.0
```



Push the domain to my Xen install:

```
oot@baye:~# sudo xl migrate guest1 10.1.1.189
root@10.1.1.189's password: 
migration target: Ready to receive domain.
Saving to migration stream new xl format (info 0x3/0x0/1411)
Loading new save file <incoming migration stream> (new xl fmt info 0x3/0x0/1411)
 Savefile contains xl domain config in JSON format
Parsing config from <saved>
xc: info: Saving domain 28, type x86 PV
xc: info: Found x86 PV domain from Xen 4.9
xc: info: Restoring domain
xc: info: Restore successful
xc: info: XenStore: mfn 0x328657, dom 0, evt 1
xc: info: Console: mfn 0x328656, dom 0, evt 2
migration target: Transfer complete, requesting permission to start domain.
migration sender: Target has acknowledged transfer.
migration sender: Giving target permission to start.
migration target: Got permission, starting domain.
migration target: Domain started successsfully.
migration sender: Target reports successful startup.
Migration successful.
```

Checking the result on my machine:

```
root@labubuntu:/mnt/private/artem# xl list
Name                                        ID   Mem VCPUs	State	Time(s)
Domain-0                                     0  4096     4     r-----    1914.5
guest1                                       4  1024     1     -b----       0.3
```

I could ping the guest machine:

```
$ ping 10.1.1.85
PING 10.1.1.85 (10.1.1.85) 56(84) bytes of data.
64 bytes from 10.1.1.85: icmp_seq=1 ttl=64 time=0.611 ms
64 bytes from 10.1.1.85: icmp_seq=2 ttl=64 time=0.435 ms
64 bytes from 10.1.1.85: icmp_seq=3 ttl=64 time=0.467 ms
64 bytes from 10.1.1.85: icmp_seq=4 ttl=64 time=0.413 ms
64 bytes from 10.1.1.85: icmp_seq=5 ttl=64 time=0.320 ms
64 bytes from 10.1.1.85: icmp_seq=6 ttl=64 time=0.159 ms
64 bytes from 10.1.1.85: icmp_seq=7 ttl=64 time=0.397 ms
```

And attach terminal:

```
artem@labubuntu:~$ sudo xl console guest1
64 bytes from lt-in-f113.1e100.net (108.177.14.113): icmp_seq=1957 ttl=45 time=312 ms
64 bytes from lt-in-f113.1e100.net (108.177.14.113): icmp_seq=1958 ttl=45 time=48.4 ms
64 bytes from lt-in-f113.1e100.net (108.177.14.113): icmp_seq=1959 ttl=45 time=43.9 ms
64 bytes from lt-in-f113.1e100.net (108.177.14.113): icmp_seq=1960 ttl=45 time=51.2 ms
64 bytes from lt-in-f113.1e100.net (108.177.14.113): icmp_seq=1961 ttl=45 time=45.5 ms
```



#### Now don’t even shut them down while migrating... (hot-migration, live-migration) 

I was in the console of the guest machine, first checked ip:

```
root@guest1:~# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:16:3e:42:40:89 brd ff:ff:ff:ff:ff:ff
    inet 10.1.1.85/24 brd 10.1.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::216:3eff:fe42:4089/64 scope link 
       valid_lft forever preferred_lft forever
```



Start pinging google from inside the guest:

```
root@guest1:~# ping google.com
PING google.com (108.177.14.100) 56(84) bytes of data.
64 bytes from lt-in-f100.1e100.net (108.177.14.100): icmp_seq=1 ttl=45 time=46.6 ms
64 bytes from lt-in-f100.1e100.net (108.177.14.100): icmp_seq=2 ttl=45 time=45.2 ms
64 bytes from lt-in-f100.1e100.net (108.177.14.100): icmp_seq=3 ttl=45 time=60.0 ms
```



Also ping the machine from the outside:

```
artem@labubuntu:~$ ping 10.1.1.85
PING 10.1.1.85 (10.1.1.85) 56(84) bytes of data.
64 bytes from 10.1.1.85: icmp_seq=1 ttl=64 time=0.133 ms
64 bytes from 10.1.1.85: icmp_seq=2 ttl=64 time=0.319 ms
64 bytes from 10.1.1.85: icmp_seq=3 ttl=64 time=0.416 ms
64 bytes from 10.1.1.85: icmp_seq=4 ttl=64 time=0.415 ms

```





Migrate the machine to Gaspar's Xen:

```
artem@labubuntu:~$ sudo xl migrate guest1 10.1.1.143
root@10.1.1.143's password: 
migration target: Ready to receive domain.
Saving to migration stream new xl format (info 0x3/0x0/1411)
Loading new save file <incoming migration stream> (new xl fmt info 0x3/0x0/1411)
 Savefile contains xl domain config in JSON format
Parsing config from <saved>
xc: info: Saving domain 2, type x86 PV
xc: info: Found x86 PV domain from Xen 4.9
xc: info: Restoring domain
xc: info: Restore successful
xc: info: XenStore: mfn 0x137bdb, dom 0, evt 1
xc: info: Console: mfn 0x137bda, dom 0, evt 2
migration target: Transfer complete, requesting permission to start domain.
migration sender: Target has acknowledged transfer.
migration sender: Giving target permission to start.
migration target: Got permission, starting domain.
migration target: Domain started successsfully.
migration sender: Target reports successful startup.
Migration successful.
```



I checked that it was running on the Gaspar's machine:

```
root@baye:~# xl list
Name                                        ID   Mem VCPUs	State	Time(s)
Domain-0                                     0 13984     4     r-----   56497.8
guest1                                      28  1024     1     --p---      28.0
```



I was tacking pings from inside the guest to google.com

The amazing thing is that I could see the last ping packet on my host:

```
64 bytes from lt-in-f113.1e100.net (108.177.14.113): icmp_seq=23 ttl=45 time=45.3 ms
```



And the first ping packet after migration:

```
64 bytes from lt-in-f113.1e100.net (108.177.14.113): icmp_seq=24 ttl=45 time=43.9 ms
```



