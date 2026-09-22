> *This translation has been made by Claude, check out the original writeup [here](README.md).*

![Screenshot](img/Pasted%20image%2020260519094339.png)

https://tryhackme.com/room/checkmate
## The Task

> Marco Bianchi, a systems administrator, recently deployed several internal services, including a firewall console, employee portal, social platform, and SSH access to critical infrastructure. Due to tight deadlines and operational pressure, Marco reused weak, predictable, and pattern-based passwords across multiple systems.  

> Your objective is to conduct a password security assessment to identify weaknesses in Marco's authentication practices.

![Screenshot](img/Pasted%20image%2020260519094509.png)
## To the website

Got an IP, so what does the application look like? I arrive at a portal where I can enter the cracked passwords.

![Screenshot](img/Pasted%20image%2020260519095759.png)
## Level 1

The first application to take a closer look at is a firewall found on port 5001.

![Screenshot](img/Pasted%20image%2020260519095939.png)

Tried `admin:admin`, but that didn't work.

I see that the domain name `firewall.thm` is printed on the page. Maybe add it to the hosts file and fuzz a bit? That gave nothing of value

![Screenshot](img/Pasted%20image%2020260519100835.png)

Nothing that reveals what type of firewall it is (brand, model, etc.), so it's hard to google default credentials. Kicked off a Hydra run with common passwords.

`hydra -l admin -P /usr/share/seclists/Passwords/Default-Credentials/default-passwords.txt firewall.thm -s 5001 http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid" -I`

And found the right password after a second

<details>
  <summary><b>Click here to see the password</b></summary>

  `12345`
</details>
  
## Level 2

![Screenshot](img/Pasted%20image%2020260519102211.png)

An employee login on the site `jobs.thm`. The password is apparently supposed to be built from common keywords that appear on the site.

![Screenshot](img/Pasted%20image%2020260519102542.png)

We're presented with a bunch of keywords right at the header:

![Screenshot](img/Pasted%20image%2020260519102634.png)

Clicking Employee Login brings us to the login form:

![Screenshot](img/Pasted%20image%2020260519102747.png)

Thanks for the pre-filled username!

Kicked off an attack in Burp Intruder and got a hit!

<details>
  <summary><b>Click here to see the password</b></summary>

  ![Screenshot](img/Pasted%20image%2020260519103051.png)
</details>


Once logged in it looks like this:

![Screenshot](img/Pasted%20image%2020260519103340.png)

Here we find some info that might come in handy later - surname, nickname and birthday.
## Level 3

![Screenshot](img/Pasted%20image%2020260519103456.png)

![Screenshot](img/Pasted%20image%2020260519103613.png)

![Screenshot](img/Pasted%20image%2020260519105146.png)

Tried using `marky` as username along with some of the previous passwords. No hits. Ran rockyou without a hit, and even created a custom wordlist with Claude without success. Tried creating a custom wordlist with **cupp**, which put together a list of about 2500 words. Still no hit. Changed the username to `marco` (as it was on the jobs site) and then got a hit!

<details>
  <summary><b>Click here to see the password</b></summary>

  `Bianchi2495`
</details>

A bit of a sneaky nod to the nickname there, and it wasn't even part of the password. Fun! A bit odd though how `24` fits into the picture, the birthdate was **14021995** (as DDMMYYYY). Oh well.
## Level 4

![Screenshot](img/Pasted%20image%2020260519110121.png)

![Screenshot](img/Pasted%20image%2020260519110219.png)

Right-clicked the profile picture to check the filename:

![Screenshot](img/Pasted%20image%2020260519110329.png)

Before turning to Hashcat, let's try Crackstation. And that gave results:

<details>
  <summary><b>Click here to see the password</b></summary>

  ![Screenshot](img/Pasted%20image%2020260519110537.png)
</details>

## Level 5

![Screenshot](img/Pasted%20image%2020260519110632.png)

![Screenshot](img/Pasted%20image%2020260519110906.png)

Created a new wordlist with **cupp**, it landed at around 10500 words. Wasn't fully happy with the list since it created so many words that fell outside Marco's rules. Ended up turning to Claude and tailoring a wordlist entirely according to Marco's rules. It came out to about 500 words and Hydra got a hit more or less immediately.

<details>
  <summary><b>Click here to see the password</b></summary>

  ![Screenshot](img/Pasted%20image%2020260519113218.png)
</details>

That's it, happy hacking!
