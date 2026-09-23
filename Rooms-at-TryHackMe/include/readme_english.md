> *This translation has been made by Claude, check out the original writeup [here](README.md).*

![Screenshot](img/Pasted%20image%2020260504162615.png)

https://tryhackme.com/room/include

![Screenshot](img/Pasted%20image%2020260504162636.png)

> _"Even if it's not accessible from the browser, can you still find a way to capture the flags and sneak into the secret admin panel?"_

Alright, server-side attacks here we go! Two flags to grab, let's start with a good old nmap and see what we can find.
## Recon

A quick port scan against all tcp ports found 7 open. A closer scan of those 7 gave the following result:

![Screenshot](img/Pasted%20image%2020260504221302.png)

Good thing I didn't cancel the first scan too early, because before it finished it also found port 50000 open. A closer look at that gave:

![Screenshot](img/Pasted%20image%2020260504221608.png)

Oh la la. A website perhaps? Let's start by taking a closer look at it.
## Port 50000

![Screenshot](img/Pasted%20image%2020260504221844.png)

Besides the admonishing text I have the option to log in.

![Screenshot](img/Pasted%20image%2020260504222004.png)

`admin:admin` didn't work :D

Found a small comment at the end of the login page that could be of interest.

![Screenshot](img/Pasted%20image%2020260504222137.png)

The login page is called `/login.php`, so why not run a ffuf against the site and see what shows up?

Found a few nice directories:

![Screenshot](img/Pasted%20image%2020260504223619.png)

How nice that the module has been about template injection, and there just happens to be a matching folder called templates. What a coincidence :D (Turned out the module hadn't been about template injection at all, it was actually a different one...)

![Screenshot](img/Pasted%20image%2020260504223720.png)

Mostly bootstrap and jquery, but a few custom things.

`footer.php` contains that exact comment I noticed on the login page:

![Screenshot](img/Pasted%20image%2020260504223829.png)

Feels like I somehow want to inject something there at the end. Maybe?

`header.php` contains, well, the header. With menu and so on.

![Screenshot](img/Pasted%20image%2020260504224001.png)

`/javascript` gives code 403. `/uploads` on the other hand:

![Screenshot](img/Pasted%20image%2020260504224145.png)

An adorable profile picture.

<img height="200px" src="Pasted%20image%2020260504224209.png">

Ran a gobuster specifically looking for php files.

![Screenshot](img/Pasted%20image%2020260504225842.png)

## Port 25

At least 5 other ports have something to do with mail - would be strange if I couldn't find something tasty there. Here's a refresher of what I found earlier:

![Screenshot](img/Pasted%20image%2020260504221302.png)

[Pentestmonkey](https://pentestmonkey.net/tools/user-enumeration/smtp-user-enum) has a nice little tool for enumerating users over smtp. This is already installed on Kali, so what are we waiting for?

`smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/top-usernames-shortlist.txt -t $IP`

![Screenshot](img/Pasted%20image%2020260505162529.png)

Didn't give me any usernames I couldn't have guessed myself. Running with "common" names too.

Then I find `bin`, another system name, but also `charles`. Takes an insanely long time because I set a 5-second timeout on each query, and the list I'm using now has 10,000 entries... Starting **enum4linux** in parallel.

Enum4linux finished because the server doesn't allow username and password to be empty. Tried sending `charles` but that didn't work either.

**smtp-user-enum** is done

![Screenshot](img/Pasted%20image%2020260505170738.png)

So `charles` and `joshua` are definitely of interest.

Can we connect?

![Screenshot](img/Pasted%20image%2020260505163116.png)

![Screenshot](img/Pasted%20image%2020260505163204.png)

## POP3 and IMAP

I'm pumped about mail. So a hydra with my valid usernames against POP3 and IMAP?

Setting up Hydra against port 995.

Ouch, ouch, ouch, Charles...

![Screenshot](img/Pasted%20image%2020260505172630.png)

And `joshua` also had 123456. That smells like a false positive from a mile away. But worth checking.

Well I never!

![Screenshot](img/Pasted%20image%2020260505173013.png)

But

![Screenshot](img/Pasted%20image%2020260505173304.png)

![Screenshot](img/Pasted%20image%2020260505173253.png)

Completely empty inboxes...
## Port 4000

According to nmap, Node.js with Express is running. What does curl say? Oh oh, some kind of login. Checking in the browser.

![Screenshot](img/Pasted%20image%2020260505171324.png)

Well, might as well do as I'm told :D `guest:guest` it is!

![Screenshot](img/Pasted%20image%2020260505171456.png)

![Screenshot](img/Pasted%20image%2020260505173513.png)

Two fields on my profile where I can recommend activities. Meow. What I write gets added to the list above.

![Screenshot](img/Pasted%20image%2020260505173659.png)

Ran a ferox and included my `connect.sid` header and had a look around, nothing super exciting.

![Screenshot](img/Pasted%20image%2020260505174802.png)

Suspecting there could be some form of Property Pollution vulnerability in this recommend-activity form.

Trying a few different payloads without success.

However, tried the slightly simpler variant of just writing "isAdmin" and "true" in the field and submitting. Then `isAdmin` was changed to `true`. Two new pages appear:

![Screenshot](img/Pasted%20image%2020260506095817.png)

The API menu option shows:

![Screenshot](img/Pasted%20image%2020260506095907.png)

And Settings shows:

![Screenshot](img/Pasted%20image%2020260506100451.png)

Here you can upload a banner that renders on the profile page. Would be nice to upload a disguised php file that reads files on the server instead :)

Or actually, we want to set the address so that it calls the internal API of course!

Pointed the URL to the getAllAdmins API and got the following:

![Screenshot](img/Pasted%20image%2020260506101017.png)

A response with a long base64 string. Nice :)

One cyberchef later:

![Screenshot](img/Pasted%20image%2020260506101153.png)

```json
{
	"ReviewAppUsername":"admin",
	"ReviewAppPassword":"admin@!!!",
	"SysMonAppUsername":"administrator",
	"SysMonAppPassword":"S$9$qk6d#**LQU"
}
```

The first flag should be on the SysMon site, so I go to the site on port 50000 and log in with `administrator:S$9$qk6d#**LQU` and the flag is secured!

<details>
  <summary><b>Click here to see it</b></summary>

  ![Screenshot](img/Pasted%20image%2020260506101812.png)
</details>

## Flag two

It should be found by reading a secret file at `/var/www/html`

What nice things can we find on the SysMon site to start with?

To our great joy we find a URL with a parameter in the code for `/dashboard.php`. It points to `/profile.php?img=profile.png` which feels like a great entry point for LFI.

Did some manual tests in Burp to check for LFI. First - check if the code strips out `../` in an attempt to stop path traversal.

In Burp I simply tried pointing to `../profile.png` instead of just `profile.png`. The image still showed, which suggests that `../` gets stripped.

The trick is to write `....//` instead - then that disallowed combination of characters gets stripped out, but the fact is what remains after that is exactly `../` :D

Tried to chase down `/etc/passwd`, and it took a whole nine `....//` to point correctly, but it worked in the end!

![Screenshot](img/Pasted%20image%2020260506115945.png)

Tried instead to point to the secret file in `/var/www/html` and guessed it would be called `flag.txt`, but nope. How could we find the filename?

## SSH

This was quite the blunder. I'd already found valid credentials for pop3, why didn't I test them on ssh?

Turned out they could log in easy as pie, and print out `/etc/passwd` in a somewhat simpler way :D

![Screenshot](img/Pasted%20image%2020260506120246.png)

And you don't think joshua could find the secret file?

![Screenshot](img/Pasted%20image%2020260506120340.png)

<details>
  <summary><b>The flag, the flag!</b></summary>

  ![Screenshot](img/Pasted%20image%2020260506120512.png)
</details>



In fact, once inside `/var/www/html` we could print out `dashboard.php` and find the first flag that was protected behind the login :D

<details>
  <summary><b>Click here to see the shenanigans</b></summary>

  ![Screenshot](img/Pasted%20image%2020260506144956.png)
</details>

So, we actually had access to both flags already after finding the two usernames and valid passwords. There you go :)

## Final words

The first flag was found by first exploiting a Mass Assignment vulnerability in the "recommend activity" form, where I could set `isAdmin: true` on my own user. I thought the vulnerability would be about Prototype Pollution, but this was similar. After that we got access to more pages, where we found a local API and could exploit an SSRF vulnerability via a form and call the API and get login credentials in return. It feels like it was completely by the book.

We found the second flag, but it feels like it was done in a way that wasn't entirely in line with the themes the module had covered. So we checked out a few writeups to see how others solved the task.

As mentioned, we found the LFI vulnerability, but didn't really know how to exploit it to find the secret file.

[This writeup](https://medium.com/@z0diac/include-ctf-tryhackme-writeup-c36bded6d2f4) describes really nicely how you can use Log Poisoning with the help of the mail logs. In short, it's about using LFI to find the file `/var/log/mail.log`. This file generally has both read and write permissions.

By sending an email via telnet or nc to one of the identified users, `mail.log` gets filled with information about that activity. Instead of providing a valid sender address you can instead write `<?php system($_GET['cmd']); ?>` which will then execute when you point to `mail.log` via LFI.

Here's [another writeup](https://medium.com/@S1L3NT37/include-tryhackme-detailed-writeup-11c74c7ac7e1) where you can follow someone who basically used the same approach as we did, but also added log poisoning at the end.

Really interesting reading, both of them. Happy hacking!
