# AS Lab 2 - Database

#### Artem Abramov SNE19

# Prepare database 

## 1. Install a DBMS that can perform column level encryption and database level encryption (f.e MySQL).

I installed the DB with docker. For this I used the MySQL image from docker hub. 
```
docker run --name mysql-db -p 3306:3306 -p 33060:33060 -e MYSQL_ROOT_PASSWORD=artem -d mysql
```

To interact with the database I used the mysql client installed in the same container:
```
docker exec -it mysql-db bash
root@ad813693c8cc:/# mysql -uroot -p
```

## 2. Create database and populate it with data (about 5 columns and 50 000 rows in a table)

```
mysql> create DATABASE test;
```

Generated test data using http://filldb.info/ a sample  is shown below:

![](AS-Lab-2-database.assets/DeepinScreenshot_select-area_20200203003522.png)

Copied the test data into the container
```
docker cp ~/Downloads/test-02-02-2020-20-56-beta.sql mysql-db:mnt
```

Inserted data into database called `test` (inside the container):
```
# cd /mnt
# mysql -u root -p
mysql> use test
mysql> source /mnt/test-02-02-2020-20-56-beta.sql
```


# Database connection

## 3. Create password based credentials that will be used by your application to access the created database.

The credential file allows storing our sensitive username and password to allow automated access to MySQL.

Spin up  `mysql-cli` container:

```
docker run --name mysql-cli -it  mysql /bin/bash
```

On it create the credentials file with identifier  `main`. (The info about `mysql-db` container was taken from `docker inspect`):

```
 mysql_config_editor set --login-path=main --host=172.17.0.2 --user=root --password
```

Connect to db:

```
mysql --login-path=main
```

Remember to `use test` after login to select the db.

## 4. Make sure the DBMS is available on some network interface and capture the  traffic on that interface during the connection (you can use  corresponding CLI to open the connection, perform some SELECT queries  and then close the connection).

I captured traffic with wireshark listening to docker0.

I performed the following from `mysql-cli` container:

```
root@b7c99e16d613:/# mysql --login-path=main
mysql> use test
mysql> select * from test limit 10;
mysql> \q
```



## 5. Analyze the captured data:

#### Can you see the plain data and credentials? How can the attacker bypass TLS layer with your present configuration? (hint: analyze TLS handshake)

The data and credentials are not present as plaintext. 

- On connection start server advertises MySQL protocol==10 and version==8.0.19. 
- Next follows a login request from the client (user), it contains a name field, however that is legacy of MySQL protocol and in this capture the field is not filled. 
- The next message is also from the client, it is a Client Hello which starts the TLS handshake. Client  (as seen in the handshake layer)  supports up to TLSv1.2. Part of the  header is shown below:

```
Transport Layer Security
    TLSv1.2 Record Layer: Handshake Protocol: Client Hello
        Content Type: Handshake (22)
        Version: TLS 1.0 (0x0301)
        Length: 175
        Handshake Protocol: Client Hello
            Handshake Type: Client Hello (1)
            Length: 171
            Version: TLS 1.2 (0x0303)
            Random: e4747df403a07859d34cbe45cb4bd8a859a46840377630f0…
            Session ID Length: 0
            Cipher Suites Length: 60
            Cipher Suites (30 suites)
                Cipher Suite: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256 (0xc02b)
                Cipher Suite: TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 (0xc02c)
                Cipher Suite: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 (0xc02f)
                Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (0xc030)
                Cipher Suite: TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256 (0xc023)
                Cipher Suite: TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256 (0xc027)
                Cipher Suite: TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384 (0xc024)
                Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384 (0xc028)
                Cipher Suite: TLS_DHE_RSA_WITH_AES_128_GCM_SHA256 (0x009e)
                Cipher Suite: TLS_DHE_DSS_WITH_AES_128_GCM_SHA256 (0x00a2)
                Cipher Suite: TLS_DHE_RSA_WITH_AES_128_CBC_SHA256 (0x0067)
                Cipher Suite: TLS_DHE_DSS_WITH_AES_128_CBC_SHA256 (0x0040)
                Cipher Suite: TLS_DHE_DSS_WITH_AES_256_GCM_SHA384 (0x00a3)
                Cipher Suite: TLS_DHE_RSA_WITH_AES_256_CBC_SHA256 (0x006b)
                Cipher Suite: TLS_DHE_DSS_WITH_AES_256_CBC_SHA256 (0x006a)
                Cipher Suite: TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA (0xc013)
                Cipher Suite: TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA (0xc009)
                Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA (0xc014)
                Cipher Suite: TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA (0xc00a)
                Cipher Suite: TLS_DHE_DSS_WITH_AES_128_CBC_SHA (0x0032)
                Cipher Suite: TLS_DHE_RSA_WITH_AES_128_CBC_SHA (0x0033)
                Cipher Suite: TLS_DHE_RSA_WITH_AES_256_CBC_SHA (0x0039)
                Cipher Suite: TLS_RSA_WITH_AES_128_GCM_SHA256 (0x009c)
                Cipher Suite: TLS_RSA_WITH_AES_256_GCM_SHA384 (0x009d)
                Cipher Suite: TLS_RSA_WITH_AES_128_CBC_SHA256 (0x003c)
                Cipher Suite: TLS_RSA_WITH_AES_256_CBC_SHA256 (0x003d)
                Cipher Suite: TLS_RSA_WITH_AES_128_CBC_SHA (0x002f)
                Cipher Suite: TLS_RSA_WITH_AES_256_CBC_SHA (0x0035)
                Cipher Suite: TLS_DHE_RSA_WITH_AES_256_GCM_SHA384 (0x009f)
                Cipher Suite: TLS_EMPTY_RENEGOTIATION_INFO_SCSV (0x00ff)
```

- In response the server chooses to use TLSv1.2 and replies with Server Hello, Certificate, Server Key Exchange, Certificate Request and Server Hello Done records.

source: https://dev.mysql.com/doc/refman/8.0/en/encrypted-connection-protocols-ciphers.html

The server allows connections without authentication:

```
mysql> SHOW GLOBAL VARIABLES LIKE 'require_secure_transport';
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| require_secure_transport | OFF   |
+--------------------------+-------+
1 row in set (0.00 sec)
```

require_secure_transport = Whether client connections to the server are required to use some form of secure transport. When this variable is enabled, the server permits only TCP/IP connections that use SSL.

The server also accepts TLSv1.0 for communication:

```
mysql> SHOW GLOBAL VARIABLES LIKE 'tls_version';
+---------------+-----------------------+
| Variable_name | Value                 |
+---------------+-----------------------+
| tls_version   | TLSv1,TLSv1.1,TLSv1.2 |
+---------------+-----------------------+
1 row in set (0.01 sec)
```

The security hazard associated with accepting non-TLS conneciton is clear and simple. 

There is also an issue with accepting TLSv1.0 connections. The TLS handshake happens in plaintext, a man in the middle can rewrite the client hello to claim that client only supports the insecure TLSv1.0, under current configuration MySQL server will accept this insecure connection as secure and valid. Example TLSv1.0 vulnerabilities: POOL, BEAST, etc.

#### What can be done to mitigate that?

Force authentication and forbid using TLSv1.0 on the server. This values can be set dynamically **no server restart required**  on MySQL server version>=8.0.16 and can be made persistent across reboot via the commands below:

```
mysql> SET PERSIST require_secure_transport = ON;
mysql> SET PERSIST tls_version="TLSv1.1,TLSv1.2";
```

Checking the connection from client:

```
root@b7c99e16d613:~# mysql --login-path=main --tls_version=TLSv1.0
ERROR 2026 (HY000): SSL connection error: TLS version is invalid
```



## 6) Capture the connection traffic with TLS disabled and extract the content of user and password fields.

For this I disabled the require_secure_transport option in the MySQL db and then connected without SSL:

```
root@b7c99e16d613:~# mysql --login-path=main --ssl-mode=DISABLED
```

Wireshark output is below:

```
Login Request
    Client Capabilities: 0xa605
    Extended Client Capabilities: 0x01ff
    MAX Packet: 16777216
    Charset: latin1 COLLATE latin1_swedish_ci (8)
    Username: root
    Password: bac51a7f6b490d54359f9e3039f83be7391102e0b900eadc…
    Client Auth Plugin: caching_sha2_password
    Connection Attributes
```

Username: root

Password: bac51a7f6b490d54359f9e3039f83be7391102e0b900eadcdd31a9ca86041453

#### Describe the authentication scheme that is used (in which form credentials are provided, how are they stored and verified)

Authentication scheme is determined by pluggable authentication architecture of MySQL server. Architecture info: https://dev.mysql.com/doc/refman/8.0/en/pluggable-authentication.html and by using a specific plugin. Wireshark dump and MySQL config confirm use of caching_sha2_password plugin for authentication of user `root`. Plugin info: https://dev.mysql.com/doc/refman/8.0/en/caching-sha2-pluggable-authentication.html

The architecture part is simple - when connection is established MySQL finds the authentication plugin that applies for the particular user, invokes it, and returns "okay" or "reject".  If the answer is "reject", the whole connection fails.

Architecture details:
```
--> client connects to the server
--> server sends the Handshake Initialization Packet. The packet carries
    the following fields:
    * Name of the plugin the server uses (after the second part of the scramble) 
      if the CLIENT_PLUGIN_AUTH bit in server capabilities is set
--> client invokes the default (or specified) client plugin. 
    * If the plugin is not the default one (native backward compatible 
      internal authentication plugin) the client will not feed it with the 
      scramble sent in server's handshake packet.
--> client makes sure the first packet sent is the auth packet as usual 
    (and sets CLIENT_PLUGIN_AUTH).
    * If it uses non-backward compatible authentication the content of 
      the scramble_buff is defined by the plugin
    * sends plugin name as ASCIIZ string after the database name if the server
      supports plugin authentication.
--> server checks that
    * host is allowed to connect
    * user name is found in the mysql.user table or there is a default 
      proxy user that gets used instead.
    * plugin specified in the "plugin" column of the
      mysql.user table is loaded
    ** if either of the above fails - authentication fails
    * and it's the same plugin as client has sent in the auth packet
    ** if it fails, see below
    and then it calls the plugin with the received username/scramble data
--> if the client has used a wrong plugin, server requests
    the client to use the correct one:
    * a server sends a special "Use this plugin: XXX" packet
    * a client replies as requested (or disconnects)
--> now auth plugin can communicate with the client to do
    any further conversation, as necessary.
```



The `caching_sha256_password` plugin does work on both the client and the server side. To connect to the server using an account that authenticates with the `caching_sha2_password` plugin, the client can use a secure connection or an **unencrypted connection that supports password exchange using an RSA key pair**. This plugin works as follows:

- Server sends 20 bytes of random data AKA salt AKA scramble buffer (split into two chunks of 8 bytes and 12 bytes) in the inital handshake packet (AKA Server Greeting). 

- Client software XORs client's SHA(password) with scramble buffer content. I could find the algorithm for `mysql_native_password` (shown below), but no exact description of the `caching_sha256_password` :( 

  I suppose it would just use SHA256 instead of SHA1.

  ```
  SHA1( password ) XOR SHA1( "20-bytes random data from server" <concat> SHA1( SHA1( password ) ) )
  ```

  Then the client RSA encrypts that with the server's public key. Sends the result to server.

- Server decrypts the RSA and verifies the client's computations to the server's computations.

If the client does not have the public key, it can ask the server.

The server stores hash of the user's password in its `mysql.user` table:

```
mysql> select host, user, plugin, authentication_string from mysql.user;
```

It seems that the auth string is encoded, because it contains unprintable chars. This is most likely the SHA256 of the password. Interestingly the type of the column `authentication_string` is TEXT, not a binary blob.

![](AS-Lab-2-database.assets/DeepinScreenshot_select-area_20200203062948.png)

The users `root`, `old_userr`, `sha2user` were created for testing.

sources: 

1. https://dev.mysql.com/doc/internals/en/secure-password-authentication.html
2. https://dev.mysql.com/doc/internals/en/sha256.html
3. https://dev.mysql.com/doc/refman/8.0/en/caching-sha2-pluggable-authentication.html

 

#### Would an attacker be able to impersonate user if he had access to the content of database? Why?

Yes. The server stores the SHA of the user's password that is used to calculate the response to the challenge. If the attacker gains that value, what would stop him from establishing a new connection providing this value in calculations?

The Secure Password Authentication page on the MySQL website claims (https://dev.mysql.com/doc/internals/en/secure-password-authentication.html): 

```
knowning the content of the hash in the mysql.user table isn't enough to authenticate against the MySQL Server. 
```

However because the attacker can just establish a new connection and get his own random challenge, I fail to see why this claim is true.

### What alternative authentication schemes are there? Which would you prefer and why? 

There are multiple authentication plugins available:

- [6.4.1.1 Native Pluggable Authentication](https://dev.mysql.com/doc/refman/8.0/en/native-pluggable-authentication.html)
- [6.4.1.2 Caching SHA-2 Pluggable Authentication](https://dev.mysql.com/doc/refman/8.0/en/caching-sha2-pluggable-authentication.html)
- [6.4.1.3 SHA-256 Pluggable Authentication](https://dev.mysql.com/doc/refman/8.0/en/sha256-pluggable-authentication.html)
- [6.4.1.4 Client-Side Cleartext Pluggable Authentication](https://dev.mysql.com/doc/refman/8.0/en/cleartext-pluggable-authentication.html)
- [6.4.1.5 PAM Pluggable Authentication](https://dev.mysql.com/doc/refman/8.0/en/pam-pluggable-authentication.html)
- [6.4.1.6 Windows Pluggable Authentication](https://dev.mysql.com/doc/refman/8.0/en/windows-pluggable-authentication.html)
- [6.4.1.7 LDAP Pluggable Authentication](https://dev.mysql.com/doc/refman/8.0/en/ldap-pluggable-authentication.html)
- [6.4.1.8 No-Login Pluggable Authentication](https://dev.mysql.com/doc/refman/8.0/en/no-login-pluggable-authentication.html)
- [6.4.1.9 Socket Peer-Credential Pluggable Authentication](https://dev.mysql.com/doc/refman/8.0/en/socket-pluggable-authentication.html)
- [6.4.1.10 Test Pluggable Authentication](https://dev.mysql.com/doc/refman/8.0/en/test-pluggable-authentication.html)
- [6.4.1.11 Pluggable Authentication System Variables](https://dev.mysql.com/doc/refman/8.0/en/pluggable-authentication-system-variables.html)

Of real use are the SHA-256 due to improved security, over Native (SHA-1) and Caching SHA-2 authentication. The only other interesting choice is LDAP because it is cross-platform and would tie the user identity to another authoritative server (such as the user's organization).



## 7) Column level encryption:

#### Try column level encryption and see how data looks like after encryption  (f.e. create a table with an encrypted column, issue an insert  statement, then issue a SELECT statement).

Create table:

```
mysql> create table encrypted
    -> ( id int(10) unsigned not null auto_increment, name varchar(255) not null, age int(3) not null, primary key (id) ) engine = innodb default charset = latin1;
```

Check:

```
mysql> describe encrypted;
+-------+--------------+------+-----+---------+----------------+
| Field | Type         | Null | Key | Default | Extra          |
+-------+--------------+------+-----+---------+----------------+
| id    | int unsigned | NO   | PRI | NULL    | auto_increment |
| name  | varchar(255) | NO   |     | NULL    |                |
| age   | int          | NO   |     | NULL    |                |
+-------+--------------+------+-----+---------+----------------+
3 rows in set (0.00 sec)

```

Insert values:

```
mysql> INSERT INTO encrypted VALUES (2, AES_ENCRYPT('bob', UNHEX(SHA2('My secret passphrase',512))), 40);
mysql> INSERT INTO encrypted VALUES (AES_ENCRYPT('bob', 'My secret passphrase'), 40);
```

Select:

```
mysql> select * from encrypted;
+----+------------------+-----+
| id | name             | age |
+----+------------------+-----+
|  1 | ûèDPe$UZZú÷Ó |  40 |
|  2 | \åܪ»®ekàÄaMFç |  40 |
+----+------------------+-----+
2 rows in set (0.00 sec)
```





## 8) Transparent database encryption:

#### Find the actual database files and try to extract some meaningful information from them

Db files are in `/var/lib/mysql`. 

Tables:
```
./test
./test/test.ibd
./test/encrypted.ibd
```

Database config files:
```
./auto.cnf
./mysqld-auto.cnf
```

Server keys (also client keys, but they are not so interesting):
```
./server-key.pem
./ca-key.pem
./public_key.pem
./server-cert.pem
./ca.pem
./private_key.pem
```

Log files are interesting, they are binary but contain the text from all the inserts as found by:

```
strings ib_logfile0 | less
```



Extract text strings from database:

```
root@e6db7b398e04:/var/lib/mysql# strings test/test.ibd | tail -n 20
>fAuer, Hackett and PurdyWalsh
	Lubowitzbury
>tDeckow, Hodkiewicz and CristMonahan
Creminland
Grady-GottliebNicolas
Milfordview
Boyer, Dach and GislasonTreutel
Tremaineport
Koelpin-DickinsonLarkin
Blickhaven
Gutmann-StrosinSchuppe
Arielborough
Hoeger, Hermiston and StarkHammes
	Enriqueside
Lebsack, Williamson and WhiteMiller
Kuhlmanton
Buckridge-HaleyJakubowski
East Groverhaven
Stehr, Jenkins and KunzeLebsack
Bartonhaven
```



