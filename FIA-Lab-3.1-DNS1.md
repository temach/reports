# FIA Lab 3.1 – DNS1

#### Artem Abramov SNE19

May I suggest viewing this document in your browser at address: 
https://github.com/temach/innopolis_university_reports/blob/master/
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

I decided not to use the default ubuntu packages and went along with compiling from release tarball. 

Unbound source: https://nlnetlabs.nl/projects/unbound/download/
NSD source: https://www.nlnetlabs.nl/projects/nsd/download/

Before compiling unbound in order to simplify the dependency chase I looked at the runtime  dependencies for the ubuntu unbound package as shown below:

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
When compiling NSD a problem occured because the ./configure could not find libevent even though there were clearly two versions installed. 

```
checking for libevent... configure: error: Cannot find the libevent library.
You can restart ./configure --with-libevent=no to use a builtin alternative.
```

Since there was a workaround, I compiled without libevent as shown below:
```
artem@ nsd-4.2.2$ ./configure --with-libevent=no
```

And compiling unbound was successful without any extra options as shown below:
```
$ ./configure
```

The resulting config  files are placed as requested:
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


### Root Servers

Note that the file download URL needed to be updated as shown below: 
```
$ sudo wget -S -N https://www.internic.net/domain/named.cache -O /usr/local/etc/unbound/root.hints
```
(source: https://wiki.alpinelinux.org/wiki/Setting_up_unbound_DNS_server)


### Resolving localhost

For writing the unbound config most of the information I used was from:
1. unbound.conf manual page (online version: https://nlnetlabs.nl/documentation/unbound/unbound.conf/)
2. default configuration file installed into the system
3. https://nlnetlabs.nl/pipermail/unbound-users/2010-September/006748.html
4. Bind9 manual (https://www.bind9.net/bind-9.13.3-manual.pdf)
5. https://linuxconfig.org/unbound-cache-only-dns-server-setup-on-rhel-7-linux
6. https://www.tecmint.com/install-configure-cache-only-dns-server-in-rhel-centos-7/



















