> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Operation Promotion

![Screenshot](img/Pasted%20image%2020260522095925.png)

https://tryhackme.com/room/operationpromotion
## Introduction

> You are up for promotion at **Hadron Security**. Your senior lead, Mara, has handed you a solo engagement against **RecruitCorp**, a small recruiting firm with a public-facing portal. Compromise the host, capture the flags, and demonstrate that you are ready for the Penetration Tester title.

![Screenshot](img/Pasted%20image%2020260522100019.png)
## Recon

Four open ports:

![Screenshot](img/Pasted%20image%2020260522100643.png)

The company is called **RecruitCorp** and the website looks like this:

![Screenshot](img/Pasted%20image%2020260522100751.png)

I see an email address that might come in handy later: `careers@recruitcorp.thm`

Feroxbuster gave a bunch of hits:

![Screenshot](img/Pasted%20image%2020260522101123.png)

Robots.txt only confirms that the endpoint `/admin` exists

![Screenshot](img/Pasted%20image%2020260522101042.png)

On the admin page I'm met with a login:

![Screenshot](img/Pasted%20image%2020260522101145.png)

Nothing secret in the source code.

My fuzz also revealed `/admin/users` which contains:

![Screenshot](img/Pasted%20image%2020260522103416.png)

When I click on `lookup.php` I get redirected to `/admin`.

At this point it feels like I should look further into the Samba stuff in search of credentials. Trying **smbclient**

`smbclient -L //$IP -N`

![Screenshot](img/Pasted%20image%2020260522101616.png)

`smbclient //$IP/public -N`

![Screenshot](img/Pasted%20image%2020260522101735.png)

Found `README.txt` but it didn't contain anything fun

![Screenshot](img/Pasted%20image%2020260522102203.png)

Let's see what **enum4linux** can offer.

Found a username that stands out: `jford`

Also found a custom domain on port 139:

![Screenshot](img/Pasted%20image%2020260522103337.png)

And that the passwords have a poor policy and stay valid for a very long time...

![Screenshot](img/Pasted%20image%2020260522103733.png)

Given this, it might be worth running a Hydra attack against the admin login

`hydra -l jford -P /usr/share/wordlists/rockyou.txt 10.114.171.112 http-post-form "/admin/:username=^USER^&password=^PASS^:F=Invalid"`

That produced no hit.

SQL injection on the admin login?

![Screenshot](img/Pasted%20image%2020260522105910.png)

Yes indeed, that worked!

![Screenshot](img/Pasted%20image%2020260522105931.png)

And now we can access `lookup.php` judging by the form.

When I click "View own profile" I learn that `jford` has ID 1. Who has 2? (It later turned out I'm not `jford` at all - user ID 1 is `admin`)

![Screenshot](img/Pasted%20image%2020260522110141.png)

But it's going to be slow listing everyone one by one. Dump them all at once?

Couldn't help myself so I ran through a bunch manually. Interesting at `ID 7`

![Screenshot](img/Pasted%20image%2020260522110646.png)

If I follow the URL I get the following:

![Screenshot](img/Pasted%20image%2020260522110735.png)

Added the machine's IP just to see the output, got the following:

![Screenshot](img/Pasted%20image%2020260522110821.png)

What happens if you add `;id` after the IP?

![Screenshot](img/Pasted%20image%2020260522110918.png)

Found our famous `jford`

![Screenshot](img/Pasted%20image%2020260522111325.png)

But I can't read his home directory since I'm `www-data`

Kept looking and found an interesting config file

![Screenshot](img/Pasted%20image%2020260522111524.png)

Getting tired of navigating in the browser, want to set up a reverse shell instead. [Revshells.com](https://revshells.com) is my go-to and I try a few different one-liners from there.

It took a long time before I found one that worked, but it eventually landed on python3

![Screenshot](img/Pasted%20image%2020260522132826.png)

A bit nicer to navigate in the terminal

![Screenshot](img/Pasted%20image%2020260522132910.png)

Upgraded my shell so it became a bit more stable. In my shell:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

Then pressed Ctrl+Z, then:

`stty raw -echo; fg` 

Enter twice, then `export TERM=xterm` Done.

Now the question is how to escalate...

Looked around the system and eventually decided to search for the database - it must be on the system somewhere.

![Screenshot](img/Pasted%20image%2020260522152101.png)

`app.db` is highly interesting.

Reading it out with **cat** didn't work, so I opened it with **sqlite3** instead. There's the `users` table as I suspected, and voilà, the passwords can be read in plaintext.

![Screenshot](img/Pasted%20image%2020260522152231.png)

However, no user for `jford`... And the admin password only works for logging into the admin portal on the website, it doesn't work to run `su jford` with that password...

Taking a closer look at the samba stuff

![Screenshot](img/Pasted%20image%2020260522155230.png)

But that was a dead end.

By now I got tired of searching manually, so I uploaded **linpeas** to the server and let it do its job:

![Screenshot](img/Pasted%20image%2020260522161824.png)

Found something nice:

![Screenshot](img/Pasted%20image%2020260522162040.png)

A cronjob to exploit?

![Screenshot](img/Pasted%20image%2020260522162523.png)

Sockets?

![Screenshot](img/Pasted%20image%2020260522162738.png)

A bit too much info to take in, and nothing that directly screams "Hey, here's the way in!". Googled `CVE-2025-38236`, but honestly, I didn't quite follow that one...

## The password
**Onind00** took a look at the password hash we found in the database's config file, namely `$2b$10$QzkXmGndA2cQLozO3xAN6eWKrl6ZXyzhYTJNF67exOmTmN5oVSEfq`

We'd already started hashcat with rockyou.txt with no result, but by creating a new password list based on keywords found on the website (around 6000 candidates) we got a hit! With the right password we could switch user to the famous `jford`

![Screenshot](img/Pasted%20image%2020260526121156.png)

In the home directory we found `user.txt` which contained the flag!
## Next flag
Where can we find `flag.txt`? Probably in the root directory.

As it happens we're allowed to run **find** with root privileges

![Screenshot](img/Pasted%20image%2020260526121526.png)

And there we found the flag:

![Screenshot](img/Pasted%20image%2020260526121730.png)

According to GTFOBins, find can give us a root shell:

![Screenshot](img/Pasted%20image%2020260526121840.png)

Voilà!

![Screenshot](img/Pasted%20image%2020260526122048.png)

And then it was a piece of cake to read out the flag.

To think that it was such a small password that held us up for so long... Lesson learned that I'm taking with me: always try with a custom wordlist!

Happy hacking! :)
