The web now filters `'|&` preventing injection like level 9, but we can use `grep` search term like `'root' /etc/passwd` meaning to grep the word `root` from `etc/passwd`:
![[Pasted image 20260630152252.png]]

I use `''` in the search term. from what i understand is applying `''` is like telling grep to search for nothingness, 0 length string in the file `'' /etc/passwd`:
![[Pasted image 20260630152716.png]]

So `'' /etc/natas_webpass/natas11` gives us the password:
![[Pasted image 20260630152821.png]]

Lvl11 pass: VUMQDmuITOEHzhviLE5V0VG9cPMQkyxd