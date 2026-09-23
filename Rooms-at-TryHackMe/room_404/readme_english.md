> *This translation has been made by Claude, check out the original writeup [here](readme.md).*

# Room 404

https://tryhackme.com/room/hh-room404-804573bf

![Screenshot](img/Pasted%20image%2020260809182218.png)

![Screenshot](img/Pasted%20image%2020260809182243.png)
## Intro
Yeah, there's not much more to say than what's in the checklist above. I've been given an IP and a hint that port 8080 is open. A quick look with curl shows that a website is being hosted, not entirely unexpected. Let's go!
## Recon
Checked out the site in the browser, looks nice. Sent off a directory enumeration with Gobuster afterwards. Maybe that was all that was needed?

![Screenshot](img/Pasted%20image%2020260809192102.png)

Curl to that endpoint?

![Screenshot](img/Pasted%20image%2020260809192709.png)

But just pointing to `/.git` resulted in:

![Screenshot](img/Pasted%20image%2020260809192741.png)

Taking it to the browser.

Hm, whichever one I click on results in a small 404

![Screenshot](img/Pasted%20image%2020260809192916.png)

The room is called `Room 404` after all...

If I type the URL manually instead of clicking the link, I get to the right place, and it's possible to navigate through most of `/.git` both in the browser and with curl, but it's a bit cumbersome. It would be most convenient to be able to search through it locally.

`git clone` against the URL? Nope...

![Screenshot](img/Pasted%20image%2020260809205659.png)

Created a virtual environment instead and installed `git-dumper`

With git-dumper I could pull down the whole repo and work on it locally

![Screenshot](img/Pasted%20image%2020260809211727.png)

Read out `README.md` and found the flag!
