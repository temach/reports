# SSN Lab 4 - RSA
#### Artem Abramov SNE19

## 1. Create a 2048 bit RSA key pair using openssl . Write your full name in a text file and encrypt it with your private key. Using OpenSSL extract the public modulus and the exponent from the public key. Publish your public key and the base64 formatted encrypted text in your report.

source: https://en.wikibooks.org/wiki/Cryptography/Generate_a_keypair_using_OpenSSL

Creating the key was done with the command below:
```
$ openssl genpkey -algorithm RSA -out private_key.pem -pkeyopt rsa_keygen_bits:2048
...................................................+++++
......+++++
```


The generated keyfile is shown below:
```
$ cat private_key.pem 
-----BEGIN PRIVATE KEY-----
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBKcwggSjAgEAAoIBAQDVRKhTKerrL+5t
WFM1omSJ5mB/MCM2Ek/TLgRja27Kvln1Py5V2Nc0hOsrWJkV9IPhHWJup93ZUFHf
SFSddEKWnjFUZe/S53R3xAwMI0iriFgeoGirPmeWyGxoMfcuqrJcpjUH/X2tkRGV
hst0I+ZBopTWxbZ+x0iEDB35XvkPiEx5+nuwmSDVvJYShp1XnFT0zOo7d+IlUjgs
RGT18DWa9f+qQIS2ZM4vMFbStuExjEzpKWRS83xqeZtkwrxSYn719hzUOiAd11vd
mgJ4gwsR8qfgqIksPaotyRBODtHk6ct9EuUDDmDOq52piZjkM6dZfCKzbAK60ohg
FqrAm0GpAgMBAAECggEBAJrDyz8msTLhQq4ZJbFLWph0FVzXSwYnPcRrpYWcZfc9
Mbz2j6J+zJVxS9PV5Kn5GmcX0uGocicM0Y0uSTFGODhjLjsoCmTL0EpFe9SnGkpY
lFINaEe07qmOYpUuhUlZWDxB5UQaIcT9/Hu9KRHsJ2FEIazGJ8xSw/4HaQ0ZbbZl
iHsoNg0OQjeOsjxB7AjL2bph7VG9nzEC/BKkQ944kaffsQaTIj4+abFrZhhIYyDA
BpeQJAquPNLuO/y5riZmU0UyTiSC4iFR91dBxlVmpx/969OYegAvBvZdJRlF/zdA
3wzb6umxM/6FmzmBpntQa50vFJJkoGNFoGyw0GTp/00CgYEA8gYiRgdPnGrnCuQt
TGZcIFlKNbHDa/foTU9B2cTRZ7TPOaFXlyT5KzFNjEX4kjHR8Bh7TMhEPiO6NGcU
1tsj9qfNU2ZmwaHPAAXwMezINIcIc+iTg/tKfhE2rktkoARGMzm5pIUVZ7sZS/bh
RtVrSRwHgVBHLtzalMg//RwS0vcCgYEA4ZVsjuEqCumjD0Oe7iHPPrAr646iyPCS
NmhJqetqj/OXJSsz5+R+r8xdH4Ctj36onnoDYmVPcs5bHbRZ8nLa0Ay3uZVD5I7R
bIGfzgq4e6J4PVZ09xs1auDRIKUciIterWRYZbSUjwl5wOJWwrbcWtRaWPsZUKQA
UWnUo0QiyF8CgYBNOgfVsJ+GmdX/c6/AoSjkaZl0aLSizwghIxkwE819+gPlr8Xz
RKj9T04fZHRcsej7Y9R0O7MK4PuZnQrZRDPFkOXDpvlC8A7NQxfR1mxL1oqr8Zqg
kp7Ce1unYgdJyBNeZR0Hki+yWdFInA5mO9NwELGUYblVCsP0lbFVYo/fIQKBgChI
XZ1aGf7e385e/ed6RB7mqMEktG77frCWiOj98AaKsifiAh/bzr+n181DVjr0nDnI
ggg1HS3cKasxRIql9esHClvqirgqxL0CnL3XbkAtsBkL8lEaG7zdLpsMwP6lHKAL
9jkbORgd0g8em+2jZ9AUcZnIJSogTyo6os0Q4sEzAoGAKQKoULvJ2Ai26p7ZzRrx
L1V0i+PiVhgtcZHkyAeRpM5J5O5TRTcvo32Z+gc6PlWBVdmzUuiHwXXMbv2XRKaT
1kagyhMDBEQy8WuQZXilHQ5ABHfyfC1NfV289MKV6a9/7Bx/nffFisebweq4Nvxt
MGIfpTIHRCeQ/I3eDlFnpyQ=
-----END PRIVATE KEY-----
```


source: https://rietta.com/blog/openssl-generating-rsa-key-from-command/

I extracted the public key with the command below:
```
$ openssl rsa -in private_key.pem -outform PEM -pubout -out public_key.pem
writing RSA key
```

The public key is shown below:
```
$ cat public_key.pem 
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA1USoUynq6y/ubVhTNaJk
ieZgfzAjNhJP0y4EY2tuyr5Z9T8uVdjXNITrK1iZFfSD4R1ibqfd2VBR30hUnXRC
lp4xVGXv0ud0d8QMDCNIq4hYHqBoqz5nlshsaDH3LqqyXKY1B/19rZERlYbLdCPm
QaKU1sW2fsdIhAwd+V75D4hMefp7sJkg1byWEoadV5xU9MzqO3fiJVI4LERk9fA1
mvX/qkCEtmTOLzBW0rbhMYxM6SlkUvN8anmbZMK8UmJ+9fYc1DogHddb3ZoCeIML
EfKn4KiJLD2qLckQTg7R5OnLfRLlAw5gzqudqYmY5DOnWXwis2wCutKIYBaqwJtB
qQIDAQAB
-----END PUBLIC KEY-----
```

I created the file with my full name in it as shown below:
```
$ cat data.txt 
Artem Abramov
```

The task asks us to encrypt the file with the public key. Keeping in mind the following interesting stack overflow answer: https://cs.stackexchange.com/questions/59675/can-a-public-key-be-used-to-decrypt-a-message-encrypted-by-the-corresponding-pri 

Below is the command to encrypt the data with the private key:

```
$ openssl rsautl -encrypt -inkey private_key.pem -in data.txt -out data.enc
```

Below is the result of encryption:

```
$ xxd data.enc 
00000000: b13f d2a4 7ce0 3771 bd9d 194d 2360 eae9  .?..|.7q...M#`..
00000010: ff3f 30dd fcb0 f151 2a45 6a05 ba84 5500  .?0....Q*Ej...U.
00000020: 1356 1f57 0773 2b6c 29b2 5bf2 2ff0 d2c3  .V.W.s+l).[./...
00000030: 71a0 df8a aa1a cf7b 6afb ebf9 768d 658d  q......{j...v.e.
00000040: be55 bb29 d2f5 a048 03b4 3acd 8408 2037  .U.)...H..:... 7
00000050: 6a9c e870 ae58 25e5 4fb9 e1c4 c961 2450  j..p.X%.O....a$P
00000060: 2863 4a83 4324 1b7e db5f 8cb0 7049 9022  (cJ.C$.~._..pI."
00000070: 2f0a 18d2 006a 41e3 f4ea 480f 81d0 697d  /....jA...H...i}
00000080: d006 071e cf3e 797d be81 ad46 f9ba d243  .....>y}...F...C
00000090: 76a1 704e b230 d6ee cda8 b922 77a5 b0af  v.pN.0....."w...
000000a0: a8ca 7bb1 9f09 dabf a8a1 0051 3c3b c40c  ..{........Q<;..
000000b0: 83ef daad 8947 efb6 23ac 16c7 277d f641  .....G..#...'}.A
000000c0: 5f4a 38ae 2504 2bf4 c0fe 49ed 752e 46f5  _J8.%.+...I.u.F.
000000d0: b1cc d873 8611 39cc dbef 94c4 5925 2c71  ...s..9.....Y%,q
000000e0: 94bc 7e58 c144 1378 f516 f4d4 bf7f 0cbb  ..~X.D.x........
000000f0: f644 e4d6 2897 b9d6 681e bda7 cfe1 6bac  .D..(...h.....k.
```

Base64 of the encrypted data is shown below:

```
$ base64 data.enc 
sT/SpHzgN3G9nRlNI2Dq6f8/MN38sPFRKkVqBbqEVQATVh9XB3MrbCmyW/Iv8NLDcaDfiqoaz3tq
++v5do1ljb5VuynS9aBIA7Q6zYQIIDdqnOhwrlgl5U+54cTJYSRQKGNKg0MkG37bX4ywcEmQIi8K
GNIAakHj9OpID4HQaX3QBgcezz55fb6BrUb5utJDdqFwTrIw1u7NqLkid6Wwr6jKe7GfCdq/qKEA
UTw7xAyD79qtiUfvtiOsFscnffZBX0o4riUEK/TA/kntdS5G9bHM2HOGETnM2++UxFklLHGUvH5Y
wUQTePUW9NS/fwy79kTk1iiXudZoHr2nz+FrrA==
```



source: https://stackoverflow.com/questions/3116907/rsa-get-exponent-and-modulus-given-a-public-key

To extract the modulus and exponent from the public key I used the command below:
```
$ openssl rsa -pubin -inform PEM -text -noout < public_key.pem 
RSA Public-Key: (2048 bit)
Modulus:
    00:d5:44:a8:53:29:ea:eb:2f:ee:6d:58:53:35:a2:
    64:89:e6:60:7f:30:23:36:12:4f:d3:2e:04:63:6b:
    6e:ca:be:59:f5:3f:2e:55:d8:d7:34:84:eb:2b:58:
    99:15:f4:83:e1:1d:62:6e:a7:dd:d9:50:51:df:48:
    54:9d:74:42:96:9e:31:54:65:ef:d2:e7:74:77:c4:
    0c:0c:23:48:ab:88:58:1e:a0:68:ab:3e:67:96:c8:
    6c:68:31:f7:2e:aa:b2:5c:a6:35:07:fd:7d:ad:91:
    11:95:86:cb:74:23:e6:41:a2:94:d6:c5:b6:7e:c7:
    48:84:0c:1d:f9:5e:f9:0f:88:4c:79:fa:7b:b0:99:
    20:d5:bc:96:12:86:9d:57:9c:54:f4:cc:ea:3b:77:
    e2:25:52:38:2c:44:64:f5:f0:35:9a:f5:ff:aa:40:
    84:b6:64:ce:2f:30:56:d2:b6:e1:31:8c:4c:e9:29:
    64:52:f3:7c:6a:79:9b:64:c2:bc:52:62:7e:f5:f6:
    1c:d4:3a:20:1d:d7:5b:dd:9a:02:78:83:0b:11:f2:
    a7:e0:a8:89:2c:3d:aa:2d:c9:10:4e:0e:d1:e4:e9:
    cb:7d:12:e5:03:0e:60:ce:ab:9d:a9:89:98:e4:33:
    a7:59:7c:22:b3:6c:02:ba:d2:88:60:16:aa:c0:9b:
    41:a9
Exponent: 65537 (0x10001)
```

The modulus is therefore: 
```
0xd544a85329eaeb2fee6d585335a26489e6607f302336124fd32e04636b6ecabe59f53f2e55d8d73484eb2b589915f483e11d626ea7ddd95051df48549d7442969e315465efd2e77477c40c0c2348ab88581ea068ab3e6796c86c6831f72eaab25ca63507fd7dad91119586cb7423e641a294d6c5b67ec748840c1df95ef90f884c79fa7bb09920d5bc9612869d579c54f4ccea3b77e22552382c4464f5f0359af5ffaa4084b664ce2f3056d2b6e1318c4ce9296452f37c6a799b64c2bc52627ef5f61cd43a201dd75bdd9a0278830b11f2a7e0a8892c3daa2dc9104e0ed1e4e9cb7d12e5030e60ceab9da98998e433a7597c22b36c02bad2886016aac09b41a9
```


## 2. Assuming that you are generating a 1024 bit RSA key and the prime factors have a 512bit length, what is the probability of picking the same prime factor twice? Explain your answer.

sources:

1. Main source of information: https://primes.utm.edu/howmany.html
2. https://en.wikipedia.org/wiki/Prime_number_theorem
3. https://crypto.stackexchange.com/questions/1970/how-are-primes-generated-for-rsa
4. https://stackoverflow.com/questions/16091581/how-many-prime-numbers-are-there-available-for-rsa-encryption

Different implementations pick the numbers in a different manner. OpenSSL approach to RSA generates a number and then applies an algorithm (and tests) to find a nearby number that very probably a primer. Then the second number is picked in a similar manner.  (Its interesting that according to the `prime number theorem` the distance between two prime numbers near `X` is approximately `ln(x)`. )

The max number that can be written in 512bits is 2**512 which is shown below (155 digits):

```
>>> 2**512
13407807929942597099574024998205846127479365820592393377723561443721764030073546976801874298166903427690031858186486050853753882811946569946433649006084096
```

The task is essentially to find how many prime numbers are there that are smaller than this number. There are multiple approaches to this, described in  https://primes.utm.edu/howmany.html. The main ones are `pi(x)` and an approach based on the Riemann zeta function.

The simplest approximation that gives the number of primes less than or equal to `x` is as follows:

```
pi(x) = x / log(x - 1)
```

However for the final approximation I used wolfram alpha computational function `PrimePi` as shown below:

![Selection_514](SSN-Lab-4-rsa.assets/Selection_514.png)



Thus the probability of picking the same prime factor twice is about 1 of `3.778 * 10^151` as shown on the screen shot above.





## 3. Explain why using a good RNG is crucial for the security of RSA. Provide one reference to a real world case where a poor RNG lead to a security vulnerability.

source: https://eprint.iacr.org/2016/367.pdf

The RNG is crucial because it is a core component in creating the large attack surface which must stop the attacker from being able to brute force the key. If the RNG is misconfigured or its randomness has patterns then this will reduce the attack surface, meaning that the attacker can filter out certain cases, which makes brute forcing feasible. An iteresting example of reducing the attack surface is understanding that in Enigma a plaintext letter was never encrypted to itself, thus the case of `A -> A` could be thrown out which makes brute forcing easier.

Example security vulnerability: Debian specific change to the OpenSSL package made the random number generation algorithm used by OpenSSL predictable, this affects various genreated keys:  SSH keys, OpenVPN keys, DNSSEC keys, and key material for use in X.509 certificates and session keys used in SSL/TLS connections. These keys are considered weak, i.e. they are much easier to guess because the RNG algorithm gave numbers not in the full range but in a limited range, thus making a brute force attack on the keys feseable. So for example the OpenSSH packages on affected Debian systems could only generate 1 of only 262148 distinct keypairs (!!). Source: https://lists.debian.org/debian-security-announce/2008/msg00152.html

The actuall change was adding a memory initialisation code as shown here: https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=363516. Source: http://taint.org/2008/05/13/153959a.html

Another interesing link on RNG and Erlang: http://files.meetup.com/1512314/Kenji%20Rikitake%20EUG%20September%202013%20.pdf

## 4. Here you can find the modulus (public information) of two related 1024bit RSA keys. Your keys are numbered using the list at http://bit.ly/2gWJvqV . Your task is to factor them i.e. retrieve p and q . You may use any tools for this. Explain your approach.

source: 

1. https://crypto.stackexchange.com/questions/13113/how-can-i-find-the-prime-numbers-used-in-rsa
2. http://doctrina.org/The-3-Seminal-Events-In-Cryptography.html
3. http://doctrina.org/How-RSA-Works-With-Examples.html

My number is 8. Therefore the information given to me is as shown below: 

```
8, 0xb2285fef8468625d9403fb75af76e049545c6722d7ebdf8a7cebd198846d7d0dd020fe4c952ebb3852e8f2fd1f9629e02e48a823259d712b330d358b89ac47ed74b72d8e584b0c4452d7fd1b187e8497e43eb570e009fe22f16ab74a7e323eb6f805f21f5d1fd998f8ee7d5853f572c4f16b7c9be81d3712e80e6d64d775d33fL, 0xb6110db3775e13c70340fa2e4dc8eae2af48f607b39bc2e78b98e07aae543f552b0607e450e48d9e2e7cc9034166545951f7450550d315140d96e8c9750e94e5ac703afc1c69822ce953b85ab6814ad3acc9e9a34eb92f2301943703200764f99ea26ce482cd874f8660e92ab522115fce014b6964c178dee23001e1a19341efL
```

I will drop the last `L` on each number, because this is not part of the number.

Modulus 1:

```
0xb2285fef8468625d9403fb75af76e049545c6722d7ebdf8a7cebd198846d7d0dd020fe4c952ebb3852e8f2fd1f9629e02e48a823259d712b330d358b89ac47ed74b72d8e584b0c4452d7fd1b187e8497e43eb570e009fe22f16ab74a7e323eb6f805f21f5d1fd998f8ee7d5853f572c4f16b7c9be81d3712e80e6d64d775d33f
```

Modulus 2:

```
0xb6110db3775e13c70340fa2e4dc8eae2af48f607b39bc2e78b98e07aae543f552b0607e450e48d9e2e7cc9034166545951f7450550d315140d96e8c9750e94e5ac703afc1c69822ce953b85ab6814ad3acc9e9a34eb92f2301943703200764f99ea26ce482cd874f8660e92ab522115fce014b6964c178dee23001e1a19341ef
```

We are told that these keys share some information and are required to find `p` and `q`. 

Modulus `n = p * q`, therefore we can assume that for these two either:

1. p1 == p2 and q1 != q2
2. p1 != p2 and q1 == q2

In other words either p or q are equal. In other words there is a number that is part of both multiplications. To find this number we need to find the common denominator between these two numbers. Using the python's math.gcd function, the common denominator is shown below (in hexadecimal):

```
0xbd8fd34b35f33bb49a96fa7ec3b0897a92ab4a899edb07a390b7234c2ffe5b5dafd68dca68c7ffd01cc9937cc4b8eb51813182740cf438482bbae53ae663ce89
```

To find the other factor we divide each modulus by the above GCD. The second parameter for Modulus 1 is shown below (in hex):

```
0xf09954ee67c400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

The seconds parameter for Modulus 2 is shown below (in hex):

```
0xf5e0bb3ace9ec0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```





## 5. Now that you have the p and q for both keys, recreate the first public and private key using this script . Encrypt your name with the private key and post the public key and the base64 formatted encrypted data in your report.

I chose to use the first p and q, i.e. the values: 
```
0xf09954ee67c400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

and 

```
0xbd8fd34b35f33bb49a96fa7ec3b0897a92ab4a899edb07a390b7234c2ffe5b5dafd68dca68c7ffd01cc9937cc4b8eb51813182740cf438482bbae53ae663ce89
```

To get the encryption working I actually needed to specify only 2 values: `n` and `e`.  I already had `n1`, so to calculate `e`, I first calculated the value of `phi(n)` as `= (p-1) * (q-1)`, as shown below:

```
0xb2285fef84686418ffe0def3304009d5b8c820ceb3bcf244e6d7b3dccfd370ab31ef007505ac4f3b075efcb79cb62258bca3295013b7e4420bb588ac540360f256115ba7a22cc44b656905813c4f76856d54b5766124f85c6f48dcb3d001a4a2502972359738002fe3366c833b4714ae7ece7d8bf30bc7b7d4451ac5199c3178
```

Then I picked an  `e` such that `gcd(phi(n), e) = 1` , which was just done by taking the suggested value for `e` from the script `0x10001`.

At this point I realized that the script was basically doing the same thing and so I just modified it as shown below before running:

```python
__author__ = 'cdumitru'

from Crypto.PublicKey import RSA
from Crypto import Random
import gmpy
import base64

#tup (tuple) - A tuple of long integers, with at least 2 and no more than 6 items. The items come in the following order:
#RSA modulus (n).
#Public exponent (e).
#Private exponent (d)
#First factor of n (p).
#Second factor of n (q)

n = 0xb2285fef8468625d9403fb75af76e049545c6722d7ebdf8a7cebd198846d7d0dd020fe4c952ebb3852e8f2fd1f9629e02e48a823259d712b330d358b89ac47ed74b72d8e584b0c4452d7fd1b187e8497e43eb570e009fe22f16ab74a7e323eb6f805f21f5d1fd998f8ee7d5853f572c4f16b7c9be81d3712e80e6d64d775d33f
p = 0xf09954ee67c400000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
q = 0xbd8fd34b35f33bb49a96fa7ec3b0897a92ab4a899edb07a390b7234c2ffe5b5dafd68dca68c7ffd01cc9937cc4b8eb51813182740cf438482bbae53ae663ce89
e = 65537L

string_to_be_encrypted="Artem"

phi = (p - 1) * (q - 1)
d = long(gmpy.invert(e, phi))

tup = (n ,e)
key = RSA.construct(tup)

random_generator = Random.new().read
data = key.encrypt(string_to_be_encrypted,random_generator)

print key.exportKey('PEM')
print key.publickey().exportKey()
print base64.b64encode(data[0])
```



After installing the dependencies  the script output was as shown below:

```
$ python2 tmp.py 
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCyKF/vhGhiXZQD+3WvduBJVFxn
Itfr34p869GYhG19DdAg/kyVLrs4Uujy/R+WKeAuSKgjJZ1xKzMNNYuJrEftdLct
jlhLDERS1/0bGH6El+Q+tXDgCf4i8Wq3Sn4yPrb4BfIfXR/ZmPjufVhT9XLE8Wt8
m+gdNxLoDm1k13XTPwIDAQAB
-----END PUBLIC KEY-----
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCyKF/vhGhiXZQD+3WvduBJVFxn
Itfr34p869GYhG19DdAg/kyVLrs4Uujy/R+WKeAuSKgjJZ1xKzMNNYuJrEftdLct
jlhLDERS1/0bGH6El+Q+tXDgCf4i8Wq3Sn4yPrb4BfIfXR/ZmPjufVhT9XLE8Wt8
m+gdNxLoDm1k13XTPwIDAQAB
-----END PUBLIC KEY-----
Xj9WhRcNSIqgJNFAsOA5yHEhMm4wJjMdzaVRy5Paph27mmksEu6wBf58C+6o3DFS2Nl88SK/P3nDNGqs0qr1kuDKgQQwr4f/vNcmP1idc9DDtpbtK4YaYMW6YN2gKqrg7YfED++Z5R+Z9ZTJT1gK+KtIDbwJ3ocwEnh3dj94t7w=

```



My public key is shown below:

```
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCyKF/vhGhiXZQD+3WvduBJVFxn
Itfr34p869GYhG19DdAg/kyVLrs4Uujy/R+WKeAuSKgjJZ1xKzMNNYuJrEftdLct
jlhLDERS1/0bGH6El+Q+tXDgCf4i8Wq3Sn4yPrb4BfIfXR/ZmPjufVhT9XLE8Wt8
m+gdNxLoDm1k13XTPwIDAQAB
-----END PUBLIC KEY-----
```



The encrypted message (my name) is shown below after base64 encoding:

```
Xj9WhRcNSIqgJNFAsOA5yHEhMm4wJjMdzaVRy5Paph27mmksEu6wBf58C+6o3DFS2Nl88SK/P3nDNGqs0qr1kuDKgQQwr4f/vNcmP1idc9DDtpbtK4YaYMW6YN2gKqrg7YfED++Z5R+Z9ZTJT1gK+KtIDbwJ3ocwEnh3dj94t7w=
```

