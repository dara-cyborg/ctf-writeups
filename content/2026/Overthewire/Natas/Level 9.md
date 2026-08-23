![[Pasted image 20260628214424.png]]
from the look of it, it's just a normal word finding. so, let's look further into its logic
![[Pasted image 20260628214504.png]]

![[Pasted image 20260628214614.png]]
we got a series of words and doing `..` in the search:
![[Pasted image 20260628214744.png]]

![[Pasted image 20260628214858.png]]

The flaw: command injection. Because the code blindly inserts the `$key` variable directly into the terminal string, the shell interprets certain characters (like `;`, `&`, or `|`) as command separators rather than text.

I tried `; cat /etc/natas_webpass/natas9` didn't work
I tried changing directory doesn't work.
I tried command substitution doesn't work.
but piping works `; apple | cat /etc/natas_webpass/natas9`:
![[Pasted image 20260630145912.png]]

lvl10 pass: EgjlkzB6E8LJyf2Obt4q7q4ewt5ZWSNv



# Extra details on the code:
code to refer to:
```php
<?
$key = "";

if(array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}

if($key != "") {
    $command = "grep -i $key dictionary.txt"; 
    passthru($command);
}
?>
```

this is the php tag:
```php
<?
// ... code ...
?>
```

this is the `variable`. In PHP, variable starts with `$`:
```php
$key = "";
```

The `$_REQUEST` superglobal contains data from submitted forms, URL query strings, and HTTP cookies.
```php
if(array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}
```

Making sure it's not empty:
```php
if($key != "") {
    // ... do something ...
}
```

Executing OS command directly from the web. `passthru()` executes an external system command and passes the raw output directly back to the browser or output buffer:
```php
passthru("grep -i $key dictionary.txt");
```

Summarizing the flow:
- PHP checks if a search happened (`needle` exists).
- It sets `$key` to `"banana"`.
- It makes sure `"banana"` isn't empty.
- It tells the Linux server: `"Hey, run the command: grep -i banana dictionary.txt"`
- The Linux server looks through the text file, finds every line containing "banana", and PHP prints those lines out onto the website for you to see.