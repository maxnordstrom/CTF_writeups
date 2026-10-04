# Five 5 minute hacks
That's at least 25 minutes of hacking, let's go! Writeups on the TryHackMe rooms [Corridor](https://tryhackme.com/room/corridor), [Neighbour](https://tryhackme.com/room/neighbour), [Lo-Fi](https://tryhackme.com/room/lofi), [Complied](https://tryhackme.com/room/compiled) and [The Game](https://tryhackme.com/room/hfb1thegame).
## Corridor

![Screenshot](img/Pasted%20image%2020260921200953.png)

https://tryhackme.com/room/corridor

> You have found yourself in a strange corridor. Can you find your way back to where you came?

IP pointing to a website with doors, and the challenge is about IDOR vulns. IDOR, DOOR, IDOOR, get it? I think there must be a pun in there.

![Screenshot](img/Pasted%20image%2020260921202510.png)

First room from the left is empty

![Screenshot](img/Pasted%20image%2020260921202531.png)

The URL is `http://IP/c4ca4238a0b923820dcc509a6f75849b` which is the md5 hash of `1`.

First room from the right har the URL md5 hash equivalent to `8`. Doesn't really make sence since there are 13 doors... Anyway, what if I try the md5 hash of `0`? Will that take me to a hidden room?

`echo -n "0" | md5sum`

Yes it did!

![Screenshot](img/Pasted%20image%2020260921203321.png)

## Neighbour

![Screenshot](img/Pasted%20image%2020260921203531.png)

https://tryhackme.com/room/neighbour

> Check out our new cloud service, Authentication Anywhere -- log in from anywhere you would like! Users can enter their username and password, for a totally secure login process! You definitely wouldn't be able to find any secrets that other people have in their profile, right?

Another IDOR vuln, let's dig into it!

The website looks like this

![Screenshot](img/Pasted%20image%2020260921231506.png)

An by viewing source I get this neat little comment

![Screenshot](img/Pasted%20image%2020260921231544.png)

`guest:guest` takes me to this

![Screenshot](img/Pasted%20image%2020260921231616.png)

And taking a closer look at the URL it looks just like this: `http://IP/profile.php?user=guest`

Well, what if I change the parameter value to `admin`?

![Screenshot](img/Pasted%20image%2020260921231852.png)

Score. Neext!
## Lo-Fi

![Screenshot](img/Pasted%20image%2020260921232007.png)

https://tryhackme.com/room/lofi

> Want to hear some lo-fi beats, to relax or study to? We've got you covered! Navigate to the website and find the flag in the **root of the filesystem.**

This room is about path traversal and file inclusion, super!

Website looks like this

![Screenshot](img/Pasted%20image%2020260921232612.png)

I can choose different youtube videos from the library, but there's also a search bar.

By searching nothing really happens except the query string in the URL changes from `?page=sleep.php` to `?search=keyword` 

What happens when sending the query `?page=../../../../../ect/passwd`?

![Screenshot](img/Pasted%20image%2020260921233234.png)

An educated guess on where to find the flag? Just `flag.txt` in root.

`../../../../../flag.txt` was the answer.

![Screenshot](img/Pasted%20image%2020260921233358.png)

## Compiled

![Screenshot](img/Pasted%20image%2020260921233613.png)

https://tryhackme.com/room/compiled

> Download the task file and get started.

Based on the title tagline, I guess I won't be able to just run `strings` on the binary. We'll see. The main task is to find the password...

Running the binary and taking a shot at a random password gives me this:

![Screenshot](img/Pasted%20image%2020261004192736.png)

I guess strings won't help me, but it can't hurt to have a look.

Not too many strings output actually, and a couple of them are quite interesting

![Screenshot](img/Pasted%20image%2020261004192942.png)

Someone declares that **strings** is for noobs :D But then it says `password:` and `DoYouEven%sCTF`. Hm. I'll have a look in Ghidra. This is what main looks like:

![Screenshot](img/Pasted%20image%2020261004193406.png)

Reading Ghidra's C-like pseudocode isn't my best game, so Claude helped me on this one. Here's the short version:

Line 9 tells us the password must start with `DoYouEven`, and `%s` stores whatever comes after it in `local_28`. On line 15 `local_28` is compared with `_init` and the check only passes if they match. So the password is... Well, I won't spell it out for you :D

![Screenshot](img/Pasted%20image%2020261004.png)

## The Game

![Screenshot](img/Pasted%20image%2020261004200134.png)

https://tryhackme.com/room/hfb1thegame

> Cipher has gone dark, but intel reveals he’s hiding critical secrets inside Tetris, a popular video game. Hack it and uncover the encrypted data buried in its code.

Yet another binary. Will strings help me this time?

Waaaay more strings in this one... But I did actually find the flag by scrolling through the output. `strings Tetrix.exe | grep THM` would have been the wiser way to go.

![Screenshot](img/Pasted%20image%2020261004200701.png)
