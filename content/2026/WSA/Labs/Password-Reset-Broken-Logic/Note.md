**Objective**: Reset Carlos's password and access his **My Account** page.

First login as `wiener:peter`:
![[Pasted image 20260530171722.png]]
`wiener`'s my account link: `https://0a0600a004662179805d94ef007f00af.web-security-academy.net/my-account?id=wiener`

Let's try `Carlos` by click on forget password and use `Carlos`'s name:
![[Pasted image 20260530171927.png]]

Use `wiener`'s link, but change the id to `id=carlos` is not working. So, let's try resetting `wiener`'s password:
![[Pasted image 20260530172145.png]]
![[Pasted image 20260530172150.png]]

`wiener`'s password reset link: `https://0a0600a004662179805d94ef007f00af.web-security-academy.net/forgot-password?temp-forgot-password-token=ozl5pff944vvv9kcjsvsntx24mjgdi2p`.

Intercept the response of `wiener` password reset email link, and change to hidden value from `wiener` to `carlos`:
![[Pasted image 20260530174439.png]]

Then enter the new password and log in as `carlos:peter`:
![[Pasted image 20260530174506.png]]



