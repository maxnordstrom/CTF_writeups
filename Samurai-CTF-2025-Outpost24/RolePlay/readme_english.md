> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# RolePlay (Web)

![Screenshot](img/Pasted%20image%2020251215232818.png)

## Description

As you can tell from the challenge's title and description, it seems to be about roles in some way. That is, based on which role you're assigned, you have access to different things. I couldn't be entirely sure though, since the description doesn't spell it out explicitly, so I just jumped to the assigned URL and started investigating.

## Initial Recon

The description gave a link that led to a website where you could log in or create an account.

![Screenshot](img/Pasted%20image%2020251215153250.png)

I created an account and logged in. I was met with the following message:

![Screenshot](img/Pasted%20image%2020251215153422.png)

So I understood that I had probably been assigned a role and that I somehow needed to make myself **admin** to find the flag. I did a bit of basic research:

- Checked the source to see if there was anything interesting in the code.
	- I could establish that the source only contained HTML, some styling, and in the script tag only JS handling the animation in the background (text jumping around).
- Checked whether any additional files were being loaded.
	- Bootstrap is loaded in - not relevant.
	- Something called `<anonymous code>` that I don't really know what is

	  ![Screenshot](img/Pasted%20image%2020251215153832.png)
	  
- No additional JS files were loaded.
- Checked cookies
	- There was a cookie here named `JWT_COOKIE`

	  ![Screenshot](img/Pasted%20image%2020251215153957.png)

To see what properties my JWT had I decoded it on **jwt.io**

![Screenshot](img/Pasted%20image%2020251215154130.png)

What I could establish was that it uses the algorithm **ES256** (Elliptic Curve), and that my user had the role **user**. I assumed that's the role I want to turn into **admin**.

## JWT Tampering

By capturing my GET request in Burp I could see my JWT, adjust its value on **jwt.io** and then send a new request with my new payload using Burp Repeater.

### Alg none

I tried the most basic hack - changing **alg** to **none**. However I got the same response as when I first logged in.

![Screenshot](img/Pasted%20image%2020251215154615.png)

I also tried simply changing the role from user to admin, but that generated the same response.

### Algorithm Confusion Attack

I read up and found that you can try performing an [Algorithm Confusion Attack](https://portswigger.net/web-security/jwt/algorithm-confusion). Somewhat similar to when I set `alg` to `none`, but instead of switching to `none` you set a different algorithm that produces a signature at the end. The article I link to above mentions how you can switch from **RS256**, which uses a private and a public key, to **HS256** which is a symmetric algorithm and therefore only uses *one* key. That is, the same key for encrypting and decrypting.

The idea is that you switch from **RS256** to **HS256**, and use the *public* key for **RS256** as the "secret key" once you've switched to **HS256**. Following along? I'm barely following myself :D

Step 1 for this to work is to find the public key. It can be found at a number of different well-known endpoints, but a feroxbuster against the site unfortunately showed no such endpoints. The next place to look is the SSL certificate.

There I found what I assumed was the public key for **RS256**, though I was never entirely sure that was the case. Oh well, you have to try, here goes nothing!

![Screenshot](img/Pasted%20image%2020251215191137.png)

At the bottom of the image are the keywords I want - both *public key*, *Elliptic Curve* and *256*.

The tool I wanted to use was [jwt_tool](https://github.com/ticarpi/jwt_tool), and for it to work I needed the public key in pem format. So I couldn't just copy/paste from the information above.

By using **openssl** I could download the public key and save it to a file: `openssl s_client -connect roleplay.appsec.nu:443 -showcerts 2>/dev/null | openssl x509 -pubkey -noout > public_key.pem`. After that I could check the key with `openssl pkey -in public_key.pem -pubin -text` and compare that against what I saw in the information about the SSL certificate.

![Screenshot](img/Pasted%20image%2020251215192823.png)

After that I used **jwt_tool** to create myself a new JWT with alg set to **HS256**, role to **admin** and the public key as the **secret**. Once that was done I sent my new JWT along in the request via Burp. Did it work? Not at all :D

I tried tons of roles, even though deep down I was convinced it would be exactly role admin...

## The Breakthrough

You'd think this was the first attempt, but no. I had done exactly the same steps as above one evening earlier, and this time together with my teammates. One failure could be user error. Two failures? Then it's probably not the right path...

Back to basics. **Onind00** read the instructions again. Carefully. He got the rest of us to read them again, even more carefully. We noticed it said something about *psychic abilities*. Maybe you should switch role to something more spiritual?

**Onind00** checked the source of the site again. Noticed a revealing comment at the very bottom of the page:

![Screenshot](img/Pasted%20image%2020251215193436.png)

One google search later on OpenJDK 17 and JWT gave us the holy grail. It turned out there's a CVE called **Psychic Signatures**. Oh boy. Here we'd tried manual stuff, and there was a CVE the whole time?? [CVE-2022-21449](https://nvd.nist.gov/vuln/detail/cve-2022-21449)

## Psychic Signatures

We immediately started reading up in various places. NIST didn't explicitly state the name Psychic Signatures, but I found [a blog post](https://www.securecodewarrior.com/article/psychic-signatures) that covered exactly this. I read the post and got stuck on the exploit section:

![Screenshot](img/Pasted%20image%2020251215194113.png)

I hopped over to Burp and edited my JWT with **role admin**, wiped out the signature and added `MAYCAQACAQA` instead and sent it off. Did it work? Yes :D

<details>
  <summary><b>Click to see the image</b></summary>

  ![Screenshot](img/Pasted%20image%2020251215194301.png)
</details><br>

Jumped over to the CTF portal and submitted the flag. Error. What? That can't be true. Pressed **Render** in Burp instead and saw that the raw response contained some garbage characters - here's the real flag:<br>

<details>
  <summary><b>Click to see the flag</b></summary>

  ![Screenshot](img/Pasted%20image%2020251215194447.png)
</details>

## Closing words

A challenge that on paper shouldn't be that hard, but wow, how much time it took us. It really comes down to knowing what to do and how. Next time I/we face JWT challenges, we'll have a couple of methods to turn to, nice.

This flag is yet another proof that teamwork is dreamwork. I had driven myself into a rut and it took a teammate to nudge me sideways and rethink.

This flag was really great to bring home in the end :)
