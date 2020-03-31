# OT Lab 1 - Outside Threats

Team: Vasili Zyabkin, Vadim Rakhmatulin, Artem Abramov

This report was written by Artem Abramov.



Company infrastructure as a team, but chosen tasks remain individual

## Infrastructure

This is described in the video.

## Individual Task

I choose a relatively fresh vulnerability in Webmin: https://www.exploit-db.com/exploits/47230

A detailed explanation and exploit details can be found here: https://www.1337pwn.com/how-to-hack-webmin-1-920-using-metasploit-remote-code-execution/

### Setting up Proof of Concept

Download snapshot of vulnerable webmin from https://www.exploit-db.com/exploits/47230

Extract the files. Before installing make sure to install perl CPAN module `Time::Local`, this can be done with:

```
cpan Time::Local
```

If cpan is not found, then check your perl version and install corresponding perl-modules package:

```
# perl --version
This is perl 5, version 28, subversion 1 (v5.28.1) built for x86_64-linux-gnu-thread-multi
(with 61 registered patches, see perl -V for more detail)

# apt-get install perl-modules-5.28
```

After installing missing perl module run:

```
./setup.sh
```

Answer questions with default, one of them will set admin password.



Connect to webmin and enable a feature:

```
Prompt users with expired passwords to enter a new one
```

It is found under Webmin -> Webmin Configuration -> Authentication



### Exploiting it

Quick metasploit guide: https://medium.com/cyberdefendersprogram/kali-linux-metasploit-getting-started-with-pen-testing-89d28944097b



Get KaliLinux with pre-installed metasploit. Run msfconsole and search for webmin. The vulnerability is referred to as:

```
exploit/linux/http/webmin_backdoor
```

Set the options as shown on the screenshot below:

![](OT-Lab-1-Webmin-Metasploit.assets/Screenshot%20from%202020-03-31%2023-29-02.png)



For setting LHOST use the KaliLinux eth0 interface.

For setting RHOSTS use /32 mask

Before running the vulnerability test that target is indeed reachable and vulnerable with "check" command:
``` 
> check
[+] 192.168.1.6:10000 - The target is vulnerable
```

Then exploit it:

![](OT-Lab-1-Webmin-Metasploit.assets/Screenshot%20from%202020-03-31%2023-01-43.png)



This is a root shell to remote machine, from which further work can be done.