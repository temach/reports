## Project SSN + AN

sources:

1. https://wiki.opendaylight.org/view/OpenDaylight_OpenFlow_Plugin:_TLS_Support
2. https://ryu.readthedocs.io/en/latest/tls.html



## Useful Information

*Diffie-Hellman*  is a key exchange algorithm whereas *RSA* is an asymmetric encryption algorithm. 

RSA is actually *two* algorithms, one for asymmetric encryption, and one for digital signatures. They are two distinct beast; although they share the same core mathematical operation and format for keys, they do different things in different ways. 

Diffie-Hellman is shown below:

![2019-11-24_at_18:54:35_screenshot](project-ssn-an.assets/2019-11-24_at_185435_screenshot.png)



RSA is shown below:

![2019-11-24_at_18:52:21_screenshot](project-ssn-an.assets/2019-11-24_at_185221_screenshot.png)





DH is a key exchange algorithm, it is used to derive the session key  that's later used in the symmetric encryption. DH by itself is not  authenticated, so while you can derive the session key without a MITM  being intercepted, there's no guarantee that the guy on the other end  that you're deriving keys together with is whoever they say they are.  Not being able to authenticate the other party means that you'd still be  vulnerable to MITM anyway.

RSA is an asymmetric encryption algorithm. RSA can be used as a key  exchange algorithm too (without DH), however the RSA key exchange algorithm lacks the property of Forward Secrecy. Forward Secrecy is the  property that an attacker that recorded your encrypted communication and  then later obtained your private key, they shouldn't be able to decrypt  the previously recorded communication. However, RSA can be used for  signing and authentication.

DH (is normally) used together with RSA so that you get a communication channel that are both authenticated and is forward secure.

Look at the TLS 1.3 proposal completely abandons RSA key exchange (RSA authentication is still there) and uses DH.

Asymmetric encryption and key exchange are somewhat equivalent: with asymmetric encryption, you can do a key exchange by virtue of generating a random symmetric key (a bunch of random bytes) and encrypting that  with the recipient's public key. (In other words: DH is more or less equivalent to using public key encryption on a random message). The downside to using public key encryption is that the secret has to be transmitted over the network (although it is encrypted). With DH the secret **never** gets transmitted over the network. 



Diffie-Hellman Key Exchange
--------------------------

**Problem:** We have a symmetric encryption scheme and want to communicate. We don't want anybody else to have the key, so we can't say it out loud (or over a wire).

**Solution/Mechanics:** 

1. We each pick a number, usually large, and keep it a secret, even from each other. I'll pick $x$, and you'll pick $y$.
2. We agree on two more numbers, both prime, which anybody can know. We'll call them $g$ and $n$.
3. I'll calculate $g^x\bmod n$ and tell you my answer. You'll calculate $g^y\bmod n$ and tell me your answer.
4.  Our shared key is $g^{xy}\bmod n = (g^y\bmod n)^x\bmod n = (g^x\bmod n)^y\bmod n$.

RSA Asymmetric Encryption
-------------------------

**Problem:** I want anybody to be able to encrypt a message, but I'm the only one who can decrypt it. I don't want to share decryption keys with anybody.

**Solution/Mechanics:** 

1. I choose two large primes, $p$ and $q$. 

2. I calculate $n$ such that $pq = n$.

3. I calculate $t$ such that $(p-1)(q-1) = t$.

4. I choose an integer $e$ that is both less than $t$ and coprime with $t$.

5. I find $d$ such that $d$ is the multiplicative inverse of $e$ modulo $t$.

6. I release $(e, n)$ as my public key, and retain $(d, n)$ as my private key.

7. When you communicate with me, you treat your message as a large number $m$. The ciphertext $c$ is given by $c = m^e\bmod n$, and decrypted by $m = c^d\bmod n$.



## Useful bits

When one interface is connected to cloud, but packets get send via another interface (that's not connected) you can shut the annoying interface off with command below:

```
ip link set dev eth0 down
```



## Setting up the controller and the switch

Updating:

```
apt-get update
```



Install python3

```
apt-get install python3
```



Install pip

```
apt-get install python3-pip
```



#### Controller

Install Ryu (make sure to use version 9.4 of `tinyrpc` package)

```
pip3 install ryu
```



#### Creating a certificate

source: https://www.linux.com/tutorials/creating-self-signed-ssl-certificates-apache-linux/



Create a CSR (this also generates a new private key `privkey.pem`):

```
# openssl req -new > new.cert.csr
Generating a RSA private key
...........................................+++++
.........................+++++
writing new private key to 'privkey.pem'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:Ru
State or Province Name (full name) [Some-State]:Tatarstan
Locality Name (eg, city) []:Inno
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Inno
Organizational Unit Name (eg, section) []:SNE
Common Name (e.g. server FQDN or YOUR name) []:artem
Email Address []:artem

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:artem
```



Transform the `privkey.pem` key into common RSA format, now the file `new.cert.key` acts as the private key of the Certificate Authority:

```
# openssl rsa -in privkey.pem -out new.cert.key
Enter pass phrase for privkey.pem:
writing RSA key
```



Sign the certificate request, this creates a valid certificate with one signature:

```
# openssl x509 -in new.cert.csr -out new.cert.cert -req -signkey new.cert.key -days 365
Signature ok
subject=C = Ru, ST = Tatarstan, L = Inno, O = Inno, OU = SNE, CN = artem, emailAddress = artem
Getting Private key
```



At this point there are the following files in current folder:

```
-rw-r--r-- 1 root root 1249 Nov 24 16:46 new.cert.cert
-rw-r--r-- 1 root root 1050 Nov 24 16:45 new.cert.csr
-rw------- 1 root root 1675 Nov 24 16:46 new.cert.key
-rw------- 1 root root 1854 Nov 24 16:45 privkey.pem
```



The file `new.cert.cert` is the signed certificate. It is shown below:

````
-----BEGIN CERTIFICATE-----
MIIDbTCCAlUCFFZkfG4x5hivYZJHVnBfIYhopw28MA0GCSqGSIb3DQEBCwUAMHMx
CzAJBgNVBAYTAlJ1MRIwEAYDVQQIDAlUYXRhcnN0YW4xDTALBgNVBAcMBElubm8x
DTALBgNVBAoMBElubm8xDDAKBgNVBAsMA1NORTEOMAwGA1UEAwwFYXJ0ZW0xFDAS
BgkqhkiG9w0BCQEWBWFydGVtMB4XDTE5MTEyNDE2NDYxMVoXDTIwMTEyMzE2NDYx
MVowczELMAkGA1UEBhMCUnUxEjAQBgNVBAgMCVRhdGFyc3RhbjENMAsGA1UEBwwE
SW5ubzENMAsGA1UECgwESW5ubzEMMAoGA1UECwwDU05FMQ4wDAYDVQQDDAVhcnRl
bTEUMBIGCSqGSIb3DQEJARYFYXJ0ZW0wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAw
ggEKAoIBAQDA6nrEDq9B2yADid8yW/rmEm1+mDgGOO4CFcpFXUP5ZikTXBv3R1eQ
Hcoct11bUU6c9glpB3oeK09rBJ0+IjHY1mev0wgC7olFDEoXpW6SnAOSH2dUv2v+
zvrparBM0TKlvymdM3rtYdghiLrifCbhjqgUto1AOFYNk2erZmYreyMWBv9LWMLO
25ZwgmLFdxJYf5tFd89tOVHdG/+kN4kD/hH+5AF8a92CIUbFKIARO2zLVnR8d4Tu
45JZD7fFBPYURND9CyC6E9bKDxA5MktBfFiMa41sEo8HCNgrUgdtkP5NBcqp7SIm
l2nwFJ3jDfjdheg+KO+qirF6r5H2uueZAgMBAAEwDQYJKoZIhvcNAQELBQADggEB
AFY/DLAIXXPyHywsSUUeMivprATa3WFdWzaZObzj5UKJHeXS/culdUWvGnno0J2G
A4jyqdLyBGmlTdvlcrbv7x433kY1UtS4uvaM81WzavYlVnSkNFWY2vF/PSJTv4OC
QgV+cNvjxtrmUaQkhsUYOekYsbfnkgn1yEOu5KzjOjOZmUAFsUPpWkGkqUn68Xgh
/MgOzM3F2tj16LHCqpaKUwpJGt8ho+WTRvqLLqVO6j2SXLufVue/wnG5srD/msep
KVtG4QvFeZ7bCQRjjI3FOBT6zazIwl9qyAz2KxwH6+J7u+QicR7ywIU+AxJURBNB
gfVkH8AKYDSTz7CzXlkckOw=
-----END CERTIFICATE-----
````



The CA's private key is shown below for completeness:

```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAwOp6xA6vQdsgA4nfMlv65hJtfpg4BjjuAhXKRV1D+WYpE1wb
90dXkB3KHLddW1FOnPYJaQd6HitPawSdPiIx2NZnr9MIAu6JRQxKF6VukpwDkh9n
VL9r/s766WqwTNEypb8pnTN67WHYIYi64nwm4Y6oFLaNQDhWDZNnq2ZmK3sjFgb/
S1jCztuWcIJixXcSWH+bRXfPbTlR3Rv/pDeJA/4R/uQBfGvdgiFGxSiAETtsy1Z0
fHeE7uOSWQ+3xQT2FETQ/QsguhPWyg8QOTJLQXxYjGuNbBKPBwjYK1IHbZD+TQXK
qe0iJpdp8BSd4w343YXoPijvqoqxeq+R9rrnmQIDAQABAoIBAQCtErBJLJN3FcvG
4O5pE7QorOem55F7o5+EJ2BskpvlKA453C0Kz+py1Qb6dFdBEdThZKpUm+ynHFwH
uwAHOk5WJySmzUEmwDJOzBO82yI0Tdu2QI5HreSJmc/pkVnR0O0LXlFLQ2BJj7qD
e1OO1atcOMv6qTxYHTK2w0QCtb97yU74j9w9BXVMbMf5jCOqIQlIKYcunAzBO9hb
5vyrjsya/eqRvibmDmSZ+T2QznR52o/Ho89TpCCKpyU/gXoyq/IdpEWTGAa/+A/C
p/dNKlaHGpluJusgZOe2iS/AhwSq3cCIRRc2VdhFQmrwffTKfE5VHCDlV2mZ5h/a
kbmIofgBAoGBAP42jiqr+JmqbvqoBMlE3G70Ol2VD66uftkjOhygSe2QCbg1BI1u
Vt4B28CWWRuD5IKoRRfcax6cOOrhPCshZZ7zlf5lZI5qik2NlCMZmS8Eh1D+VshC
QF/k3gH5V2CMwho7EcHXng+WNY62xOn5mK9RLC0OXavAnrK9Ef9RPaHBAoGBAMJF
n3BNO9INpvxYeC2MFAgIzlgBGV/1xzsOfi1ESHvnjUrmNK9oiS/hrA4W7rgGwG6T
xP8aVGE+ImQA93iybynxeYhZn/IDMKUtZiPo4waZ2ZjEgU1TaSEiIOq9I7bFWYsi
a//I+B6nmouIRbNIdV7t2lLFPZtfOIgL7dfq6YvZAoGANQRjufqLqmClLJmGBgPa
hTliQsonryKtSdPS9s1nHaiAnm5QOL7eVwbEMfIpO1fSfH1ji4povMWQCUxYJBae
/vuGOLwksJ7/oRT55CgejnO/VOD2cGWAz38H3hmGY5h3ksjm9ypkyJNnMBBmGPX5
Jfcj43Gvj3ZoopZT/o+O7wECgYB4ofDyNBM8SNvzACCH796UaptuzCfsjVqMUmCJ
TUhRICR9aNs4PIpLYC+NhDHxtceSIlIGMlxJEeMbA/qnSO18NwTHZRfLmK2BM+Kj
0739YFVc5lCAKpYyS9XyzUW6/Wqv2c0ERM/FgT+W+ySEwi6XpsyV7wJlU0hwOjAT
jjw9aQKBgDoUHFwoqyTUrdtNItf/aWmIoH4kCNkGs66k0ykLWKkSWf3fbQz9eBOl
Mi/2CfSxFa3k22TG0+gOyzVu5pT6sz5rPcrn6g6QDJwLwtdyg6o6TYtJg/BrN4IX
T2PLeZMIYpeST3E44WKQF0RcYevoc/Ypm54ktBUU+F0WvUSCyEnt
-----END RSA PRIVATE KEY-----
```





#### Switch

This is done on the new switch

```
apt-get update
```

```
apt-get install openvswitch-switch
```



##### Certificate for the switch

This is done on the controller. The controller must have the PKI structure in place (`/var/lib/openvswitch/pki`).

```
cd /etc/openvswitch/
ovs-pki req+sign scXX switch
```

Where `XX` can be any number. In the example below it is `1`.



The following files are present:

```
-rw-r--r-- 1 root root 4360 Nov 25 20:18 sc1-cert.pem
-rw------- 1 root root 1679 Nov 25 20:18 sc1-privkey.pem
-rw-r--r-- 1 root root 3841 Nov 25 20:18 sc1-req.pem
```



Copy the `sc1-cert.pem` and `sc1-privkey.pem` from the controller to the switch directory `/etc/openvswitch`.  Also copy the controller's root CA certificate (the one that was generated with `ovs-pki init`). In the controller it is located at `/var/lib/openvswitch/pki/controllerca/cacert.pem`, put it into `/etc/openvswitch/` on the switch.

This directory will now look as below:
```
-rw-r--r-- 1 root root 4360 Nov 25 20:21 sc1-cert.pem
-rw-r--r-- 1 root root 1679 Nov 25 20:24 sc1-privkey.pem
-rw-r--r-- 1 root root 4201 Nov 25 20:30 cacert.pem
```


Configure the switch to use SSL:
```
ovs-vsctl set-ssl /etc/openvswitch/sc1-privkey.pem /etc/openvswitch/sc1-cert.pem /etc/openvswitch/cacert.pem
```

In case of error like `ovs-vsctl: unix:/var/run/openvswitch/db.sock: database connection failed (No such file or directory)`, execute the command below:

```
/usr/share/openvswitch/scripts/ovs-ctl start
```

Then configure the bridge:
```
ovs-vsctl add-br br0
```

Finally set the controller and specify that switch must use SSL to connect:
```
ovs-vsctl set-controller br0 ssl:10.0.0.1:6633
```











##### Certificate for the switch

source: 

1. http://docs.openvswitch.org/en/latest/howto/ssl/#switch-key-generation-with-a-switch-pki-more-secure

Generate a certificate signing request:

```
# ovs-pki req sc
sc-req.pem	Sun Nov 24 17:38:28 UTC 2019
	fingerprint 593f3573e55f9983801b837b63a3d08e481b58c2
```



The files are generated as below:

```
-rw------- 1 root root 1675 Nov 24 17:38 sc-privkey.pem
-rw-r--r-- 1 root root 3835 Nov 24 17:38 sc-req.pem
```



Copy `sc-req.pem` to controller and sign it with Controller's key:





#### 