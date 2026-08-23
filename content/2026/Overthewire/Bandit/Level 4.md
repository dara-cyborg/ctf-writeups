This level seems to be teaching us the combination of `-` and how to read multiple files at once.
![[Pasted image 20260628112618.png]]

I tried with cat alone, we can see the password, but in an ugly way:
![[Pasted image 20260628113235.png]]

I want to read each file and end it with a newline. so, `awk` comes in. `awk` automatically break lines into variables:
![[Pasted image 20260628113335.png]]

Nothing... we have to specify `1` that tells `awk` to print every lines it reads:
![[Pasted image 20260628113643.png]]

So, level 5 password: 6C7h9GD8M6ai5nr7wo1RonrzFjj9yIrG