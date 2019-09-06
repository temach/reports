# FIA Lab 3.1 – DNS1

#### Artem Abramov SNE19

May I suggest viewing this document in your browser at address: 
https://github.com/temach/innopolis_university_reports/blob/master/FIA-Lab-3.1-DNS1.md
Unfortunately rendering the document to PDF breaks some long lines and crops images.

## Task 1 - Downloading and Installing a Caching Nameserver

I have an odd table number and so working with Unbound+NDS.

### 1.1 - Validating the Download

#### Why is it wise to use a signature to check your download?

The signature is used to verify the integrity and the authenticity of the download. If only a hash is used then this can be used to verify the integrity of the data, but can not verify the authenticity.

#### Download the BIND tarball (also if you are doing the Unbound+NSD part) and check its validity using one of the signatures.

I went to the website and downloaded two files:
```bash
-rw-rw-r--  1 artem artem    6313555 Sep  5 01:23  bind-9.14.5.tar.gz
-rw-rw-r--  1 artem artem        833 Sep  5 01:23  bind-9.14.5.tar.gz.sha512.asc
```

One is the BIND9 tarball and the other is the signature.

Then I calculated the SHA512 hashsum over the tarball as shown below:

```bash
$ sha512sum bind-9.14.5.tar.gz
1b18eda5dea639f9b34e1c41b534704b0d5f64c036b766c9cfccf9bbeb586ce4ea7f0d098a5b2747e88aa403e48ad8ae0b6e560e93348f0dc7616f914671d084  bind-9.14.5.tar.gz
```

Then to check that the signature is valid I downloaded their public key from https://www.isc.org/201920pgpkey/ and imported it into GPG.

```bash
$ gpg --import tmp.txt 
gpg: key 74BB6B9A4CBB3D38: 3 signatures not checked due to missing keys
gpg: key 74BB6B9A4CBB3D38: public key "Internet Systems Consortium, Inc. (Signing key, 2019-2020) <codesign@isc.org>" imported
gpg: Total number processed: 1
gpg:               imported: 1
gpg: no ultimately trusted keys found
```

Then on to actual verification as shown below::
```bash
$ gpg --verify bind-9.14.5.tar.gz.sha512.asc bind-9.14.5.tar.gz
gpg: Signature made Thu 15 Aug 2019 12:50:38 MSK
gpg:                using RSA key AE3FAC796711EC59FC007AA474BB6B9A4CBB3D38
gpg: Good signature from "Internet Systems Consortium, Inc. (Signing key, 2019-2020) <codesign@isc.org>" [unknown]
gpg: WARNING: This key is not certified with a trusted signature!
gpg:          There is no indication that the signature belongs to the owner.
Primary key fingerprint: AE3F AC79 6711 EC59 FC00  7AA4 74BB 6B9A 4CBB 3D38
```

The signature is a detached signature, so its provided as a separate piece of information to download. To be honest in this particular case the signature does not provide much of an improvement over the hashsum, because the signature can not be traced to a root certificate that is trusted. Therefore the trust chain does not actually get build. 

(source: https://www.isc.org/pgpkey/ and https://www.gnupg.org/gph/en/manual/x135.html).


#### Which kind of signature is the best one to use? Why?

Signing is supposed to provide integrity and authentication of the tarball. The hashsum is normally expected to provide only the integrity of the data.

In this particular case however the chain of trust does not get build. Therefore I can indeed verify that the tarball was signed with a key pair that is published on the official ISC website. I know that the website is official because the access is via HTTPS. However I still do not know if the key pair published on the official ISC website is the key pair that the ISC main administator used to sign the tarball. Maybe the webserver was compromised, the tarball was substituted and the key pair was substituted as well, so the tarball was signed with a malicious private key and the malicious public key was published on the official website.

What I want to verify  is that I have the tarball that was signed (or hash calculated) by the ISC main administator.

The chain of trust certificates can not be build (as the gpg output shows above), therefore the signature suffers exactly the same problem as the hashsum: an attacker can change the tarball and the hashsum/public key and it would not be noticed automatically. 

The signing mechanism can fix this if the ISC main administrator would get his public key signed by other authorities. Then the chain of trust could be build backwards until the certificates that I actually trust could be reached. In this case when the tarball would be substituted on the webserver,  gpg would rightfully warn that the chain of trust could not be build and the file is not to be trusted.

An interesting note: The are ways to significantly increases the level of security by distibuting the hashsum/public key via another channel. For example if the hashsum for the tarball was published on the official ISC twitter account, then by matching the hashsums I could be much more confident that the tarball was the one uploaded by the ISC main administrator, because its quite unlikely that the twitter account was compromised simultaneously with the webserver.

(interesting links:: https://www.gnupg.org/gph/en/manual/x334.html and https://serverfault.com/questions/569911/how-to-verify-an-imported-gpg-key and
https://crypto.stackexchange.com/questions/5646/what-are-the-differences-between-a-digital-signature-a-mac-and-a-hash)

### 1.2 - Documentation & Compiling

I decided not to use the default Ubuntu packages and went along with compiling from release tarball. 

Unbound source: https://nlnetlabs.nl/projects/unbound/download/
NSD source: https://www.nlnetlabs.nl/projects/nsd/download/

Before compiling unbound in order to simplify the dependency chase I looked at the runtime  dependencies for the Ubuntu unbound package as shown below:

```
$ apt-cache depends unbound
unbound
  Depends: adduser
  Depends: dns-root-data
  Depends: lsb-base
  Depends: openssl
    openssl:i386
  Depends: unbound-anchor
  Depends: libc6
  Depends: libevent-2.1-6
  Depends: libfstrm0
  Depends: libprotobuf-c1
  Depends: libpython3.6
  Depends: libssl1.1
  Depends: libsystemd0
  Suggests: apparmor
  Enhances: munin-node
```
(source: https://askubuntu.com/questions/80655/how-can-i-check-dependency-list-for-a-deb-package)

Then I installed the dependencies, before starting the actual compilation.
When compiling NSD a problem occurred because the ./configure could not find libevent even though there were clearly two versions installed. 

```
checking for libevent... configure: error: Cannot find the libevent library.
You can restart ./configure --with-libevent=no to use a builtin alternative.
```

Since there was a workaround, I compiled without libevent as shown below:
```
artem@ nsd-4.2.2$ ./configure --with-configdir=/usr/local/etc/nsd --with-libevent=no

```

And compiling unbound was successful without any extra options as shown below:
```
$ ./configure
```

The resulting configuration files are placed as requested:
```
artem@ unbound-1.9.3$ tree /usr/local/etc/nsd/
/usr/local/etc/nsd/
└── nsd.conf.sample

0 directories, 1 file
artem@ unbound-1.9.3$ tree /usr/local/etc/unbound/
/usr/local/etc/unbound/
└── unbound.conf

0 directories, 1 file
```


## Task 2 - Configuring and Testing

### Why are caching-only name servers still useful?

In Linux there is no DNS caching done at the kernel level. It is the job of user-space tools. The distributions that use systemd ship with systemd-resolved.service which acts as a local DNS server, can cache the result of the DNS lookups and is normally enabled by default. The existence of a system-wide DNS cache speeds up the resolution of commonly used domain names and saves network traffic.
(source: https://unix.stackexchange.com/questions/28553/how-to-read-the-local-dns-cache-contents and https://fedoraproject.org/wiki/Changes/Default_Local_DNS_Resolver)

Appart from systemd-resolved there are other lightweight caching DNS servers such as `dnsmasq` and `NSCD (Name Service Caching Daemon)` (source: https://www.addictivetips.com/ubuntu-linux-tips/flush-dns-cache-on-linux/) 

To check out how useful the systemd-resolved.service actually is  we can check out its statistics. Recent articles use `resolvectl statistics` that  is only available with systemd version 239, whereas on my Ubuntu the systemd version is 237, so I have to use the older command as shown below: 
```
$ systemd-resolve --statistics 
DNSSEC supported by current servers: no

Transactions
Current Transactions: 0
  Total Transactions: 232

Cache
  Current Cache Size: 60
          Cache Hits: 122
        Cache Misses: 112

DNSSEC Verdicts
              Secure: 0
            Insecure: 0
               Bogus: 0
       Indeterminate: 0
```

(source: https://askubuntu.com/questions/1149364/why-is-resolvectl-no-longer-included-in-bionic-and-whats-the-alternative and https://www.ctrl.blog/entry/systemd-resolved.html)

This are my usage statistics for the last 30 minutes, because previously I had disabled the systemd-resolved.service. Every cache hit is a success. To check the effectiveness of the caching server we can also compare the time to resolve a DNS query (using `drill` utility from `ldnsutils` package) from the remove server vs the cache.

Below is the result of sending two queries to a government website in New Zeland, note the `Query time` parameter that is 0 for the second query because the query result is cached:

![artem@artem-209-HP-EliteDesk-800-G1-SFF: ~_170](FIA-Lab-3.1-DNS1.assets/artem@artem-209-HP-EliteDesk-800-G1-SFF%20_170.png)



### Root Servers

Note that the file download URL needed to be updated as shown below: 
```
$ sudo wget -S -N https://www.internic.net/domain/named.cache -O /usr/local/etc/unbound/root.hints
```
(source: https://wiki.alpinelinux.org/wiki/Setting_up_unbound_DNS_server)


### Resolving localhost

Running unbound would conflict with systemd-resolved.service. There was no need to use the systemd-resolved.service, so it was disabled and the symbolic link to `/etc/resolv.conf` was removed as shown below:

```bash
$ systemctl disable systemd-resolved.service 
Removed /etc/systemd/system/dbus-org.freedesktop.resolve1.service.
Removed /etc/systemd/system/multi-user.target.wants/systemd-resolved.service.
artem@ unbound-1.9.3$ 
artem@ unbound-1.9.3$ file /etc/resolv.conf 
/etc/resolv.conf: symbolic link to ../run/systemd/resolve/stub-resolv.conf
artem@ unbound-1.9.3$ 
artem@ unbound-1.9.3$ sudo rm /etc/resolv.conf
```

(source: https://askubuntu.com/questions/907246/how-to-disable-systemd-resolved-in-ubuntu)

For writing the unbound config most of the information I used was from:
1. Manual page for unbound.conf (online: https://nlnetlabs.nl/documentation/unbound/unbound.conf/)
2. Default configuration file installed in the system
3. Cache size configuration: https://nlnetlabs.nl/pipermail/unbound-users/2010-September/006748.html
4. Bind9 manual (https://www.bind9.net/bind-9.13.3-manual.pdf)
5. https://wiki.archlinux.org/index.php/unbound
6. https://linuxconfig.org/unbound-cache-only-dns-server-setup-on-rhel-7-linux
7. https://www.tecmint.com/install-configure-cache-only-dns-server-in-rhel-centos-7/

After writing the configuration file I tried to check it:
```
$ unbound-checkconf 
[1567718050] unbound-checkconf[2478:0] fatal error: user 'unbound' does not exist.
```
This is an installation problem (because `make install` does not create the `unbound` user) , for simplicity I decided to run the server as root instead of adding a new user.

The resulting configuration for unbound is given below:
```
$ cat unbound.conf
server:
	verbosity: 1
	num-threads: 1
	interface: 0.0.0.0
	interface: ::0
	port: 53
	access-control: 0.0.0.0/0 allow
	access-control: ::0/0 allow
	root-hints: "/usr/local/etc/unbound/root.hints"
	username: ""
python:
remote-control:
```

Finally checking the config was successful:
```
$ unbound-checkconf 
unbound-checkconf: no errors in /usr/local/etc/unbound/unbound.conf
artem@ unbound$ echo $?
0
```

#### Why do the programs return a result value?

POSIX standard specifies that a POSIX compliant program should return 0 to indicate that it has done its job and terminated successfully and any number from 1 to 255 to indicate error.
This allows writing shell scripts that can execute programs and can control the execution flow depending on the program return code.


## Task 3 - Running and Improving the Name Server

Running unbound failed the first time to open port 53 as shown below:. 
```
$ sudo unbound -d -vvv
[1567718988] unbound[2832:0] notice: Start of unbound 1.9.3.
[1567718988] unbound[2832:0] debug: creating udp4 socket 0.0.0.0 53
[1567718988] unbound[2832:0] debug: creating tcp4 socket 0.0.0.0 53
[1567718988] unbound[2832:0] error: can't bind socket: Address already in use for 0.0.0.0 port 53 (len 16)
[1567718988] unbound[2832:0] fatal error: could not open ports
```

I investigated with netstat as shown below:
```
artem@ unbound$ netstat -lnt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 192.168.122.1:53        0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN     
tcp6       0      0 ::1:631                 :::*                    LISTEN     
```

Then I realized that I accidentally enabled libvirt network and so I turned it off as shown below:
```
artem@ unbound$ virsh net-list
 Name                 State      Autostart     Persistent
----------------------------------------------------------
 default              active     yes           yes

artem@ unbound$ virsh net-destroy default
Network default destroyed

artem@ unbound$ virsh net-list
 Name                 State      Autostart     Persistent
----------------------------------------------------------

artem@ unbound$ netstat -lnt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN     
tcp6       0      0 ::1:631                 :::*                    LISTEN  
```

Then starting unbound was successful as shown below:
```
$ sudo unbound -d -vvv
[1567719043] unbound[3018:0] notice: Start of unbound 1.9.3.
[1567719043] unbound[3018:0] debug: creating udp4 socket 0.0.0.0 53
[1567719043] unbound[3018:0] debug: creating tcp4 socket 0.0.0.0 53
[1567719043] unbound[3018:0] debug: creating udp6 socket :: 53
[1567719043] unbound[3018:0] debug: creating tcp6 socket :: 53
[1567719043] unbound[3018:0] debug: switching log to syslog
```

### Show the changes you made to your configuration to allow remote control.

Enabling log file and unbound-control in unbound.conf, the resulting config is shown below:

```
$ cat unbound.conf
server:
	verbosity: 1
	num-threads: 1
	interface: 0.0.0.0
	interface: ::0
	port: 53
	access-control: 0.0.0.0/0 allow
	access-control: ::0/0 allow
	root-hints: "/usr/local/etc/unbound/root.hints"
	username: ""
	logfile: "/var/log/unbound.log"
python:
remote-control:
	control-enable: yes
	control-interface: /var/tmp/unbound-control.pipe
```

Because I am using the named pipe mechanism for communication between unbound-control and the unbound server, there is no need to setup the TLS certificates and keys.


### What other commands/functions does rndc/unbound-control provide?

The unbound-control can be used to remotely administer the unbound server. The communication happens over SSL. There is a program to setup a selfsigned certificate and keys used in communication.  

There are a number of useful commands that the server understands such as:
1. start/stop the server
2. view statistics
3. view cache
4. create/read/update/delete a zone
5. change server mode: caching, caching+forwarding, master, slave

unbound-control provides a way to access the functionallity above remotely.

### What do you need to put in resolv.conf (and/or other files) to use your own name server?

As I have already noted above in the beginning of the section `Resolving localhost`, I have disabled management of `/etc/resolv.conf` by systemd-resolved.service. Now in order to make use of the unbound name server I need `/etc/resolv.conf` file to have the contents as below:
```
$ cat /etc/resolv.conf 
nameserver 127.0.0.1
```

Commands to start unbound server, check that its listening on port 53 and find its process id are shown below:
```
artem@ unbound$ sudo unbound-control start
artem@ unbound$ netstat -lnt
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 0.0.0.0:53              0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN     
tcp6       0      0 :::53                   :::*                    LISTEN     
tcp6       0      0 ::1:631                 :::*                    LISTEN     
artem@ unbound$ ps aux | grep unbound
root      4469  0.1  0.0  23992  5436 ?        Ss   01:56   0:00 unbound -c /usr/local/etc/unbound/unbound.conf
artem     4473  0.0  0.0  21532  1116 pts/0    S+   01:56   0:00 grep --color=auto unbound
```

Lets test that it actually provides caching. Running two consecutive queries note the `Query time` field which is 121msec for the first query and 0 msec for the second query to the same domain:

![artem@artem-209-HP-EliteDesk-800-G1-SFF: -usr-local-etc-unbound_171](FIA-Lab-3.1-DNS1.assets/artem@artem-209-HP-EliteDesk-800-G1-SFF%20-usr-local-etc-unbound_171.png)

To verify that  the speedup was due to the unbound server, lets change the default nameserver to 8.8.8.8 (Google Public DNS) and run the query again as shown below:

![root@artem-209-HP-EliteDesk-800-G1-SFF: ~_172](FIA-Lab-3.1-DNS1.assets/root@artem-209-HP-EliteDesk-800-G1-SFF%20_172.png)

The second query took as much time as the first, because the unbound server was side stepped.

## Task 4 - (Installing and) Configuring an Authoritative Nameserver

NSD has pretty good documentation, that is located in `doc/README` in the repository (github repo: https://github.com/NLnetLabs/nsd). The compilation and installation was done above in section `1.2 - Documentation & Compiling`.

Create a new `nsd` user as below to complete the installation (we disable interactive login):
```
sudo useradd -r -s /bin/false nsd
```
(source: https://askubuntu.com/questions/29359/how-to-add-a-user-without-home)

The resulting config for NSD is shown below:
```
$ cat nsd.conf
server:
	server-count: 1
	ip-address: 0.0.0.0
	ip-address: ::0
	port: 53
	verbosity: 1
	username: nsd
	logfile: "/var/log/nsd.log"
remote-control:
	control-enable: yes
	control-interface: /var/tmp/nsd-control.pipe
```

For simplicity the communication with nsd-control is handled through a named pipe.
It was also necessary to change ownership of the NSD database file as shown below (error was shown in the NSD log file):
```
$ sudo chown nsd:nsd -R /var/db/nsd/
```

Server finally started with the following line in the log file `/var/log/nsd.log`:
```
[2019-09-06 01:41:36.080] nsd[25509]: notice: nsd starting (NSD 4.2.2)
[2019-09-06 01:41:36.083] nsd[25511]: info: zone std9.os3.su read with success
[2019-09-06 01:41:36.116] nsd[25511]: notice: nsd started (NSD 4.2.2), pid 25510
```

Running the server with a sample zone file was uneventful:
```
$ nsd-checkconf -z std9.zone nsd.conf
$ sudo nsd-control start
```

### What information do you need to give to the TAs so they can implement the delegation?

My machine will be the authoritative server for the std9.os3.su zone. I need to provide the TA (who supposedly control the os3.su zone) with information at which IP address my authoritate server can be found - `188.130.155.42`. They will add non-authoritative records (glue records) to their zone which will point to the IP address of my nameserver.
(source: https://serverfault.com/questions/309622/what-is-a-glue-record)

### Now create a forward mapping zone file for your domain.

The zone file must contain:
1. 2 MX records. Make sure that mail for your domain is delivered to your
own public IP. We will use the second MX record later on.
2. 4 A or AAAA records. Use your imagination.
3. 2 CNAME records.

The resulting zone config `/usr/local/etc/nsd/std9.os3.su.zone`is shown below:
```
; source: http://www.thecave.info/create-dns-record-for-subdomain
; source: http://www.thecave.info/add-new-zone-to-bind-dns-server
; source: https://en.wikipedia.org/wiki/Zone_file
; IPv6 source: https://www.oreilly.com/library/view/dns-and-bind/9781449308025/ch01.html
; Zone file for std9.os3.su
;
$TTL 3600
; Zone configuration (SOA) record
std9.os3.su.       IN      SOA     ns0.std9.os3.su. admin.std9.os3.su. (
                        2019090600  ; Serial
                        10800       ; Refresh
                        3600        ; Retry
                        604800      ; Expire
                        38400 )     ; Negative Cache TTL
; Nameserver records
std9.os3.su.		IN      NS      ns0.std9.os3.su.

; dual stack server, but no IPv6 for the machine in lab
ns0.std9.os3.su.       	IN      A       188.130.155.42
lab.std9.os3.su.       	IN      A       188.130.155.42
mail.std9.os3.su.       IN      A       188.130.155.42

ansible.std9.os3.su.   	IN      A       185.22.153.49
                        IN      AAAA    2a00:b700:0000:0000:0000:0000:0006:0220
tst.std9.os3.su.       	IN      A       68.183.92.166
                        IN      AAAA    2400:6180:0100:00d0:0000:0000:08c4:9001

; Other records
www.std9.os3.su.   	IN      CNAME   notes.std9.os3.su.
notes.std9.os3.su. 	IN      CNAME   temach.github.io.

; Mail
std9.os3.su.                       IN      MX      10                                  mail.std9.os3.su.
std9.os3.su.                       IN      MX      20                                  ansible.std9.os3.su.
```

The resulting nsd.conf is shown below:
```
server:
	server-count: 1
	ip-address: 0.0.0.0
	ip-address: ::0
	port: 53
	verbosity: 2
	username: nsd
	logfile: "/var/log/nsd.log"
remote-control:
	control-enable: yes
	control-interface: /var/tmp/nsd-control.pipe
zone:
 	name: "std9.os3.su"
 	zonefile: "std9.os3.su.zone"
```

Showing the forward mapping zone file in the log file using the command below (the name of the zone file showed up in logs only after verbosity was set to 4):

![artem@artem-209-HP-EliteDesk-800-G1-SFF: -usr-local-etc-nsd_173](FIA-Lab-3.1-DNS1.assets/artem@artem-209-HP-EliteDesk-800-G1-SFF%20-usr-local-etc-nsd_173.png)


The next step was setting the nameserver to localhost in /etc/resolv.conf to interrogate the NSD server.
Request SOA:
```
$ drill std9.os3.su
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 13282
;; flags: qr aa rd ; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 0 
;; QUESTION SECTION:
;; std9.os3.su.	IN	A

;; ANSWER SECTION:

;; AUTHORITY SECTION:
std9.os3.su.	3600	IN	SOA	ns0.std9.os3.su. admin.std9.os3.su. 2019090600 10800 3600 604800 38400

;; ADDITIONAL SECTION:

;; Query time: 0 msec
;; SERVER: 127.0.0.1
;; WHEN: Fri Sep  6 04:49:40 2019
;; MSG SIZE  rcvd: 75
```

Request double CNAME:
```
drill www.std9.os3.su
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 21150
;; flags: qr aa rd ; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 0 
;; QUESTION SECTION:
;; www.std9.os3.su.	IN	A

;; ANSWER SECTION:
www.std9.os3.su.	3600	IN	CNAME	notes.std9.os3.su.
notes.std9.os3.su.	3600	IN	CNAME	temach.github.io.

;; AUTHORITY SECTION:

;; ADDITIONAL SECTION:

;; Query time: 0 msec
;; SERVER: 127.0.0.1
;; WHEN: Fri Sep  6 04:50:01 2019
;; MSG SIZE  rcvd: 83
```

Request with additional info:
```
$ drill ansible.std9.os3.su.
;; ->>HEADER<<- opcode: QUERY, rcode: NOERROR, id: 28530
;; flags: qr aa rd ; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1 
;; QUESTION SECTION:
;; ansible.std9.os3.su.	IN	A

;; ANSWER SECTION:
ansible.std9.os3.su.	3600	IN	A	185.22.153.49

;; AUTHORITY SECTION:
std9.os3.su.	3600	IN	NS	ns0.std9.os3.su.

;; ADDITIONAL SECTION:
ns0.std9.os3.su.	3600	IN	A	188.130.155.42

;; Query time: 0 msec
;; SERVER: 127.0.0.1
;; WHEN: Fri Sep  6 04:51:08 2019
;; MSG SIZE  rcvd: 87
```


### What important requirement is not yet met for your subdomain?
Currently the server claims to be the authoritative server for std9.os3.su. However there needs to be some way of actually proving this claim. The proof would be that the higher level DNS server delegates the std9.so3.su. zone to my server with IP `188.130.155.42`. 

We can see that the domain does not resolve currently:
```
$ dig ns std9.os3.su +trace +nodnssec

; <<>> DiG 9.11.3-1ubuntu1.8-Ubuntu <<>> ns std9.os3.su +trace +nodnssec
;; global options: +cmd
.			258385	IN	NS	i.root-servers.net.
.			258385	IN	NS	h.root-servers.net.
.			258385	IN	NS	j.root-servers.net.
.			258385	IN	NS	k.root-servers.net.
.			258385	IN	NS	f.root-servers.net.
.			258385	IN	NS	c.root-servers.net.
.			258385	IN	NS	g.root-servers.net.
.			258385	IN	NS	e.root-servers.net.
.			258385	IN	NS	l.root-servers.net.
.			258385	IN	NS	b.root-servers.net.
.			258385	IN	NS	m.root-servers.net.
.			258385	IN	NS	a.root-servers.net.
.			258385	IN	NS	d.root-servers.net.
;; Received 239 bytes from 8.8.8.8#53(8.8.8.8) in 30 ms

su.			172800	IN	NS	a.dns.ripn.net.
su.			172800	IN	NS	b.dns.ripn.net.
su.			172800	IN	NS	d.dns.ripn.net.
su.			172800	IN	NS	e.dns.ripn.net.
su.			172800	IN	NS	f.dns.ripn.net.
;; Received 352 bytes from 192.58.128.30#53(j.root-servers.net) in 17 ms

os3.su.			345600	IN	NS	ns.os3.su.
os3.su.			345600	IN	NS	ns2.os3.su.
;; Received 107 bytes from 193.232.128.6#53(a.dns.ripn.net) in 68 ms

os3.su.			1800	IN	SOA	os3.su. p\.braun.innopolis.ru. 1566480002 3600 900 1209600 1800
;; Received 96 bytes from 62.210.16.8#53(ns2.os3.su) in 66 ms
```

And we just do the query for the A record:
```
$ dig std9.os3.su +trace +nodnssec

; <<>> DiG 9.11.3-1ubuntu1.8-Ubuntu <<>> std9.os3.su +trace +nodnssec
;; global options: +cmd
.			256418	IN	NS	m.root-servers.net.
.			256418	IN	NS	j.root-servers.net.
.			256418	IN	NS	b.root-servers.net.
.			256418	IN	NS	f.root-servers.net.
.			256418	IN	NS	i.root-servers.net.
.			256418	IN	NS	h.root-servers.net.
.			256418	IN	NS	g.root-servers.net.
.			256418	IN	NS	c.root-servers.net.
.			256418	IN	NS	e.root-servers.net.
.			256418	IN	NS	a.root-servers.net.
.			256418	IN	NS	d.root-servers.net.
.			256418	IN	NS	l.root-servers.net.
.			256418	IN	NS	k.root-servers.net.
;; Received 239 bytes from 8.8.8.8#53(8.8.8.8) in 30 ms

su.			172800	IN	NS	a.dns.ripn.net.
su.			172800	IN	NS	b.dns.ripn.net.
su.			172800	IN	NS	d.dns.ripn.net.
su.			172800	IN	NS	e.dns.ripn.net.
su.			172800	IN	NS	f.dns.ripn.net.
;; Received 352 bytes from 198.97.190.53#53(h.root-servers.net) in 151 ms

os3.su.			345600	IN	NS	ns.os3.su.
os3.su.			345600	IN	NS	ns2.os3.su.
;; Received 107 bytes from 194.85.252.62#53(b.dns.ripn.net) in 27 ms

std9.os3.su.		1800	IN	A	62.210.110.7
os3.su.			1800	IN	NS	ns.os3.su.
os3.su.			1800	IN	NS	ns2.os3.su.
;; Received 123 bytes from 62.210.16.8#53(ns2.os3.su) in 71 ms
```

The IP `62.210.110.7` is actually `os3.su.` its a default responce for unknown records in the `os3.su` zone (administered by `p.braun@innopolis.ru`).

