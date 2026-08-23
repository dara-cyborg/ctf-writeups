This is level focuses on cryptography and cookie manipulation.

XOR has a very specific mathematical weakness: it is perfectly symmetrical. If you know the original unencrypted message, and you know the final encrypted output, you can easily reverse-engineer the secret key.

**The XOR Magic Rule:** If 
`Plaintext ^ Key = Ciphertext` Then 
`Plaintext ^ Ciphertext = Key`

So the `$defaultData` is `array( "showpassword"=>"no", "bgcolor"=>"#ffffff")` that gets encrypted `EGAgHwQ1IxYYMSQYGSZxTUksPFVHYDEQCC0%2FGBlgaVVIcGBDWHBgVRY%3D`.

First, let's understand the code:
```php
<?
$defaultdata = array( "showpassword"=>"no", "bgcolor"=>"#ffffff");

function xor_encrypt($in) {
    $key = '<censored>';
    $text = $in;
    $outText = '';

    // Iterate through each character
    for($i=0;$i<strlen($text);$i++) {
    $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }

    return $outText;
}

function loadData($def) {
    global $_COOKIE;
    $mydata = $def;
    if(array_key_exists("data", $_COOKIE)) {
    $tempdata = json_decode(xor_encrypt(base64_decode($_COOKIE["data"])), true);
    if(is_array($tempdata) && array_key_exists("showpassword", $tempdata) && array_key_exists("bgcolor", $tempdata)) {
        if (preg_match('/^#(?:[a-f\d]{6})$/i', $tempdata['bgcolor'])) {
        $mydata['showpassword'] = $tempdata['showpassword'];
        $mydata['bgcolor'] = $tempdata['bgcolor'];
        }
    }
    }
    return $mydata;
}

function saveData($d) {
    setcookie("data", base64_encode(xor_encrypt(json_encode($d))));
}

$data = loadData($defaultdata);

if(array_key_exists("bgcolor",$_REQUEST)) {
    if (preg_match('/^#(?:[a-f\d]{6})$/i', $_REQUEST['bgcolor'])) {
        $data['bgcolor'] = $_REQUEST['bgcolor'];
    }
}

saveData($data);
?>
```

The default data, this is similar to python dictionary:
```php
$defaultdata = array( "showpassword"=>"no", "bgcolor"=>"#ffffff");
```

The XOR encryptor:
```php
function xor_encrypt($in) {
    $key = '<censored>';
    $text = $in;
    $outText = '';

    for($i=0;$i<strlen($text);$i++) {
        $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }
    return $outText;
}
```
This function takes an input string (`$in`) and scrambles it using the secret `$key` via the **XOR (`^`)** operator.
- **`for(...)`**: It loops through the input text character by character (`$text[$i]`).
- **`$key[$i % strlen($key)]`**: This is a clever trick. Because the secret key is much shorter than the data being encrypted, the modulo operator (`%`) forces the key to "loop" back to its first character whenever it reaches the end. If the key is 4 characters long, it cycles `0, 1, 2, 3, 0, 1, 2, 3...` matching the length of the text.
- **`^` (XOR)**: It combines the bits of the text character with the bits of the key character to produce a scrambled byte, which it appends to `$outText`.

Check if the user has cookie then decrypts it, load their settings:
```php
function loadData($def) {
    global $_COOKIE;
    $mydata = $def; // Start with the default settings
```

See if `data` exists:
```php
if(array_key_exists("data", $_COOKIE)) {
        $tempdata = json_decode(xor_encrypt(base64_decode($_COOKIE["data"])), true);
```
If the cookie exists, it reverses the encoding sequence like an onion:
1. **`base64_decode`**: Converts the text cookie back into raw encrypted binary bytes.
2. **`xor_encrypt`**: Runs the bytes through the XOR function again. (XORing an encrypted string with the key a second time perfectly decrypts it back to original text).
3. **`json_decode(..., true)`**: Converts that decrypted JSON text back into a usable PHP array.

```php
if(is_array($tempdata) && array_key_exists("showpassword", $tempdata) && array_key_exists("bgcolor", $tempdata)) {
            if (preg_match('/^#(?:[a-f\d]{6})$/i', $tempdata['bgcolor'])) {
                $mydata['showpassword'] = $tempdata['showpassword'];
                $mydata['bgcolor'] = $tempdata['bgcolor'];
            }
        }
    }
    return $mydata;
}
```
check the decrypted data before trusting it.

```php
function saveData($d) {
    setcookie("data", base64_encode(xor_encrypt(json_encode($d))));
}
```
This is the exact opposite of `loadData`. When the server wants to save the user's settings back into their browser cookie, it:
1. **`json_encode`**: Turns the PHP settings array into a plain text string.
2. **`xor_encrypt`**: Scrambles the text using the secret key.
3. **`base64_encode`**: Converts the unreadable scrambled binary bytes into safe, printable URL characters.
4. **`setcookie`**: Sends it to your browser to be saved.

It runs the functions in order:
```php
$data = loadData($defaultdata); // Load existing cookie data (or defaults)

if(array_key_exists("bgcolor",$_REQUEST)) { // Did the user submit a new background color?
    if (preg_match('/^#(?:[a-f\d]{6})$/i', $_REQUEST['bgcolor'])) {
        $data['bgcolor'] = $_REQUEST['bgcolor']; // Update the array with the new color
    }
}

saveData($data); // Encrypt and save the updated array back to the cookie
```


To summarize the encryption flow is:
`json_encode($defaultData) --> xor_encrypt() --> base64_encode()`.

The decryption flow would be:
`base64_decode() --> xor_encrypt() --> json_decode()`

The script to find the key:
```php
<?
	$plaintext = json_encode(array("showpassword"=>"no", "bgcolor"=>"#ffffff"));
	
	$cookieraw = "EGAgHwQ1IxYYMSQYGSZxTUksPFVHYDEQCC0%2FGBlgaVVIcGBDWHBgVRY%3D";
	
	$ciphertext = base64_decode($cookieraw);
	
	$key = "";
	
	for($i=0; $i<strlen($plaintext); $i++) {
    	$key .= $plaintext[$i] ^ $ciphertext[$i];
	}
	
	echo "Raw key is $key";
?>
```
Output: `Raw key is kBSwkBSwkBSwkBSwkBSwkBSwkBZ{GvGk)`

The repeated pattern is `kBSw`.

Generate our own cookie:
```php
<?
	function xor_encrypt($in) {
	    $key = 'kBSw';
	    $text = $in;
	    $outText = '';
	
	    // Iterate through each character
	    for($i=0;$i<strlen($text);$i++) {
	    $outText .= $text[$i] ^ $key[$i % strlen($key)];
	    }
	
	    return $outText;
	}
	
	$rawData = array( "showpassword"=>"yes", "bgcolor"=>"#ffffff");
	echo base64_encode(xor_encrypt(json_encode($rawData)));
?>
```
Output: `EGAgHwQ1IxYYMSQYGSZxTUk7NgRJbnEVDCE8GwQwcU1JYTURDSQ1EUk/`

Result:
![[Pasted image 20260630185339.png]]
lvl12 pass: EAGkE8uzFTxeoTT2mMst9Xy7PX6guEng