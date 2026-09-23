> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Fowsniff CTF

![Screenshot](img/Pasted%20image%2020260427091512.png)

https://tryhackme.com/room/ctf

![Screenshot](img/Pasted%20image%2020260427091647.png)

Sounds like a good warm-up for a Tuesday morning.
## Scan

![Screenshot](img/Pasted%20image%2020260427091808.png)

![Screenshot](img/Pasted%20image%2020260427091932.png)

Okidoki - probably a live website and mail services.
## The Website
The website just says they've been hit by an attack.

![Screenshot](img/Pasted%20image%2020260428082912.png)

## Mail

On the website you can read:

![Screenshot](img/Pasted%20image%2020260428083026.png)

Would be nice to find these credentials.

Found md5 hashes on an old pastebin via google -> twitter -> pastebin -> waybackmachine. The pastebin page had actually been removed due to its content, but Tobzon came up with the brilliant idea of checking the waybackmachine.

Ran crackstation on the hashes.

![Screenshot](img/Pasted%20image%2020260427093017.png)

All but one, which I have to say is a pretty good result. Set up a wordlist with all the creds and ran Hydra against the pop3 server.

![Screenshot](img/Pasted%20image%2020260427094650.png)

![Screenshot](img/Pasted%20image%2020260427094602.png)

There were two mails on the server.

![Screenshot](img/Pasted%20image%2020260427094758.png)

Nice with a temporary password for ssh :)

![Screenshot](img/Pasted%20image%2020260427095030.png)

I don't understand much of the second mail, oh well.

Ran Hydra against ssh to find out which user the password worked for.

![Screenshot](img/Pasted%20image%2020260427095842.png)

![Screenshot](img/Pasted%20image%2020260427095827.png)

Logged in!

![Screenshot](img/Pasted%20image%2020260427095925.png)

Learned a new command for enumeration: `getent group`

Searched for files that my current group can write to:

`find / -writable -type f 2>/dev/null`

Grepped out txt files with a simple pipe `| grep txt`

![Screenshot](img/Pasted%20image%2020260427101614.png)

It turned out that we can write to the file that contains the ASCII banner shown when you log in via ssh. The script that runs on login fetches the text file, and the script runs as root.

Put in a reverse shell from pentestmonkey. It went a bit wrong at first because I had looked at a version that hadn't wrapped the IP number in quotes. But, once it was in place I got a reverse shell when I logged in via ssh.

![Screenshot](img/Pasted%20image%2020260427103711.png)

No flags to grab, just a nice feeling of owning the machine :)
