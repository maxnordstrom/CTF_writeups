> *This translation has been made by Claude, check out the original writeup [here](readme.md).*

# Silent Monitor

*2026-06-08*

![Screenshot](img/Pasted%20image%2020260608123512.png)
## Intro

#### Green Lights, Dark Corners

> CorpNet's internal network operations centre has been running quietly for years. Monitoring hosts, logging events, and keeping the infrastructure alive. Or so it seems. A tip from a disgruntled contractor suggests that someone on the NOC team has been cutting corners, leaving doors open, and hiding things in places no one thinks to look.
> 
> The portal is up. The services show green. The audit log looks clean.
> 
> But clean logs can be written by anyone.
> 
> Your job is to get in, move through the system, and find out what is really running behind the secret dashboard.

![Screenshot](img/Pasted%20image%2020260608123718.png)
## Recon

Ran Nmap against the IP and it responded with port 22 and 5050 open. Added the flag `-sV` to get some more info about port 5050 specifically and got the following response:

![Screenshot](img/Pasted%20image%2020260608124409.png)

HTTP, let's go!
### The web

![Screenshot](img/Pasted%20image%2020260608124711.png)

Nothing to click on, nothing in the source code, feroxbuster gave no hits.

Added corpnet.thm to `/etc/hosts` to see if it helped, but not really. It was a bit of a long shot.

Fuzzing for subdomains maybe? Maybe later.

After a bit more fiddling with **fuff** the endpoint `/internal` was found, nice.

![Screenshot](img/Pasted%20image%2020260609093038.png)

Tried `'OR 1=1--;` in the username field and it looks like I get redirected.

![Screenshot](img/Pasted%20image%2020260609093419.png)

Caught the request in Burp Proxy instead of Repeater and modified the request on the go. After that I reached the dashboard, logged in as the user `netops`.

![Screenshot](img/Pasted%20image%2020260609093745.png)

At `/internal/health` you're supposed to be able to ping various addresses to check status.

![Screenshot](img/Pasted%20image%2020260609094357.png)

When I enter the server's IP I get a response that looks believable, and when I enter a random private IP I get 100% packet loss, so it actually looks like it's running for real, not just for show.

It turns out we can send additional commands to the server after the ping command. This is revealed in the overview on the site:

![Screenshot](img/Pasted%20image%2020260609100430.png)

When I sent the following in Burp Repeater I got a nice hit :)

![Screenshot](img/Pasted%20image%2020260609100527.png)

What's in `secret.config`?

![Screenshot](img/Pasted%20image%2020260609100759.png)

Credentials! For SSH? Yes :) So the first flag is secured.
## Flag 2

In the home directory of `sysadmin` there's a directory called `backups`. In it there's `infrastructure.kdbx`, which is a Keepass database file, and if I remember correctly there's a program that can open them more or less painlessly.

Checked my old notes and found a note from the 2025 Christmas challenges. John the Ripper can settle the matter.

But, when I ran `keepass2john` I got the following error message:

![Screenshot](img/Pasted%20image%2020260609101749.png)

Looks like John doesn't support that format. Google says I can try keepass2john.com instead, interesting. But there I also get an error.

A bit of searching online doesn't give me any new leads on that front. Time to look at escalation another way - it's possible we don't need the keepass file after all.
### Escalation

Not allowed to run sudo

![Screenshot](img/Pasted%20image%2020260609102800.png)

`find / -type f -perm -04000 -ls 2>/dev/null` gave no interesting hits on binaries with the SUID bit set.

In `/opt` there's the directory `/netops`. We don't have permissions to it as the user `sysadmin`, but `www-data` has access, so let's take a look via the bug on the website instead. Turns out that's exactly where things like `secret.config` are located. Time to take a closer look at the rest of the files.

![Screenshot](img/Pasted%20image%2020260609103524.png)

Since our user doesn't have access to the directory, I had to copy the database to `/tmp`, then start a python server and pull it down to my local machine.

But that resulted in a 404 since `sysadmin` doesn't have permissions for the file. Start the python server as `www-data`?

![Screenshot](img/Pasted%20image%2020260609104942.png)

A bit of trouble understanding where the python server gets started, though.

![Screenshot](img/Pasted%20image%2020260609125525.png)

My friends found the root flag, but I didn't have time to focus on that, I really wanted to succeed in downloading the database :D

Tried a whole bunch of different payloads from revshells.com, but kept running into two different problems. Either they didn't work on the server, or my payloads were blocked by some WAF or similar.

Tobzon helped me establish that `&`, whether in plaintext or URL-encoded, was being blocked. So the idea came up to send a payload in base64, decode it, and then execute it. When I did this in one go - in an attempt to start a rev shell without writing anything to the server - I still failed. Could the WAF have caught the invalid character anyway?

I had figured out that I could create new files and write to them as `www-data` through the bug on the website. So a new idea took shape:

- Create a file, e.g. `/tmp/hej.txt`
- Write a payload to it in base64 via the web - no invalid characters
- Run `base64 -d /tmp/hej.txt > /tmp/rev.sh`
- Make the file executable with chmod
- Send the payload `bash /tmp/rev.sh` and catch the shell in my local listener.

Did it work? Yes! An incredible win. Now I could finally, without any trouble, download the database `netops.db`. Did it contain anything important? Absolutely not, but that didn't matter :D

![Screenshot](img/Screenshot%20from%202026-06-09%2012-31-33.png)
## The root flag

It turned out that all our detours led back to the keepass file and John the Ripper. What was needed was to download a different version of John, one that can handle the newer variant of keepass files.

Had to download and install the `bleeding-jumbo` version of John the Ripper.

The latest version of `keepass2john` came with it, so it was just a matter of running it. Handled the .kdbx file gracefully, which gave me a hash that john understands.

Ran john with rockyou and got an early hit:

![Screenshot](img/Pasted%20image%2020260609151326.png)

Used the password to open the keepass file with `KeePassXC` and there I found the password for root:

![Screenshot](img/Pasted%20image%2020260609151458.png)

SSH'd into the server as `sysadmin` (since I couldn't SSH as root?), `su root`, and then it was just a matter of grabbing the flag. Sweet!

## TL;DR

Base64-encode your future payloads ;) Happy hacking!
