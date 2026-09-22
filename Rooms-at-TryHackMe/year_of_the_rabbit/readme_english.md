> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Year of the Rabbit

https://tryhackme.com/room/yearoftherabbit

![Screenshot](img/Pasted%20image%2020260204084523.png)

![Screenshot](img/Pasted%20image%2020260204084741.png)

![Screenshot](img/Pasted%20image%2020260204084754.png)

## Initial Recon

`nmap -sS -sV 10.81.152.190 -v`

![Screenshot](img/Pasted%20image%2020260204085206.png)

Ran curl against the IP and it was the Apache2 Default Page.

![Screenshot](img/Pasted%20image%2020260204090107.png)

Ran a Feroxbuster with a simple wordlist. Nothing of direct value, just got RickRolled...

![Screenshot](img/Pasted%20image%2020260204090356.png)

Testing to look for info via anonymous login via ftp. But that didn't work.

Checked for clues in the CSS file and found one!

![Screenshot](img/Pasted%20image%2020260204091052.png)

Went to the URL and got the following alert:

![Screenshot](img/Pasted%20image%2020260204091156.png)

When you click OK you end up at a rickroll video again... But maybe there's something under the alert?

We curled the endpoint instead (to avoid JS) and got the following:

![Screenshot](img/Pasted%20image%2020260204091840.png)

Curled the next one

![Screenshot](img/Pasted%20image%2020260204092109.png)

The page has moved and I don't see what to in curl, checking the browser

Turned out I'd made a mistake, should've had the trailing slash. Now curl worked, and in the browser it looked like this:

![Screenshot](img/Pasted%20image%2020260204092618.png)

Downloaded Hot_Babe.png and ran strings, found some clues there:

![Screenshot](img/Pasted%20image%2020260204092813.png)

So now we have the right username for FTP and a number of passwords to build a wordlist with. Time to run hydra against the FTP

`hydra -l ftpuser -P wordlist.txt ftp://$IP`

![Screenshot](img/Pasted%20image%2020260204094134.png)

Logged in via FTP and found the following, downloaded it

![Screenshot](img/Pasted%20image%2020260204094601.png)

Contains brainfuck

![Screenshot](img/Pasted%20image%2020260204094634.png)

Went to https://www.dcode.fr/brainfuck-language and got the following out:

![Screenshot](img/Pasted%20image%2020260204094744.png)

`eli:DSpDiM1wAEwid`

Testing it on the FTP since that's the only login option we've found. The login succeeded and we found the following:

![Screenshot](img/Pasted%20image%2020260204095156.png)

All folders are empty, but the file `core` looks interesting. Downloading it and it's an ELF file.

![Screenshot](img/Pasted%20image%2020260204095241.png)

Ran strings but found no obvious flag - feels like we need to run the program to move forward somehow. Time to google a bit.

## User Flag

Onind00 tried using the same credentials for SSH, and that worked :D

When we ssh'd in we got the following message:

![Screenshot](img/Pasted%20image%2020260204125622.png)

Chose to search for parts of the "secret" string

`find / -type d -name "*s3cr*" 2>/dev/null`

And got the following hit:

![Screenshot](img/Pasted%20image%2020260204130325.png)

![Screenshot](img/Pasted%20image%2020260204130400.png)

Found the secret message, and we can surely use it for SSH. `gwendoline:MniVCQVhQHUNI`

![Screenshot](img/Pasted%20image%2020260204130441.png)

And we found the user flag

![Screenshot](img/Pasted%20image%2020260204130558.png)

## Root Flag

Ran `sudo -l` to see if gwendoline has any higher privileges, and the user did.

![Screenshot](img/Pasted%20image%2020260204131418.png)

Went straight to GTFOBins and checked what fun stuff you can do with **vi**

Couldn't read any files in `/root`

Could spawn a shell, but not a root shell.

We do get to run vi as *any* user, but not root.

After a bunch of searching it turned out there's a CVE called CVE-2019-14287

![Screenshot](img/Pasted%20image%2020260204133935.png)

![Screenshot](img/Pasted%20image%2020260204133922.png)

I didn't quite manage to get it to work when running it the way the blog showed though. Checked NIST instead:

![Screenshot](img/Pasted%20image%2020260204143743.png)

Here's something interesting, because if I do as I've now read on several sites, it only states that I'm not allowed to run `/usr/bin/vi` as `#-1`.

![Screenshot](img/Pasted%20image%2020260204144412.png)

But if I try to run `vi` against the exact file that's specified, the file with the user flag opens. So when I run
`sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt`

It turned out that once I've opened **vi** (and specified `-u#-1`) I should run `:!/bin/bash` to spawn a root shell.

![Screenshot](img/Pasted%20image%2020260204145833.png)

And the flag was found in the root directory

![Screenshot](img/Pasted%20image%2020260204145915.png)

I tried adjusting my command based on what I'd read on GTFOBins

![Screenshot](img/Pasted%20image%2020260204150750.png)

But every time I try to add `-c ':!/bin/bash'` at the end I'm asked for a password and get an error message saying I don't have permission. So it seems you need to do it in several steps - first open the file with sudo, then spawn a shell. Instead of writing `:!/bin/bash` you can also write `:shell`
