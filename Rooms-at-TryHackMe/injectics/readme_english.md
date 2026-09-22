> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Injectics

![Screenshot](img/Pasted%20image%2020260424171310.png)

https://tryhackme.com/room/injectics

The last room in the module *Injection Attacks* in the *Web Application Pentesting* path. It only consists of two questions:

![Screenshot](img/Pasted%20image%2020260424171413.png)

I've started the machine and got an IP address, so let's go!
## Initial recon
Starting with a quick and broad nmap scan via `nmap -p- $IP -v`

Pretty quickly I see that port 80 is open, so before the scan is fully done I open the browser to see if there's a website. The room is about injection and logging into an admin panel after all :)

There is a website.

![Screenshot](img/Pasted%20image%2020260424172152.png)

There are subpages for events, athletes, contact etc. but they're all disabled. The only subpage that works is *Login*. Spicy.

![Screenshot](img/Pasted%20image%2020260424172336.png)

I'm supposed to get the first flag by logging in as admin. Possibly you can find clues if you manage to log in as a regular user, but there doesn't seem to be any way to register.

The login page for admin is basically visually identical, just missing the button.

![Screenshot](img/Pasted%20image%2020260424172916.png)

What shows that you're trying to log in as admin is that the URL points to `adminLogin007.php` instead of `login.php`

Time to look for vulnerabilities related to injection.
## Scanning for injection vulns

Before launching some fancy scanner I test a few basic tricks, like `' OR 1=1--`

Started Burp so I can easily send off a few variants with Repeater.

However it would be nice to have a valid email address for this, so I dig through to see if there's a fun comment in the source code or another file. Turned out not to be a bad idea - on the start page I could find this:

![Screenshot](img/Pasted%20image%2020260424173647.png)

I'll assume `dev@injectics.thm` is a valid email address and write my payload in the password field

The `' OR 1=1--` trick only responds with invalid email or password though.

![Screenshot](img/Pasted%20image%2020260424174041.png)

I test a few different variants, both in the username field and the password field. Doesn't seem to find anything that works though. Here are a few I've tested:

![Screenshot](img/Pasted%20image%2020260424175213.png)

![Screenshot](img/Pasted%20image%2020260424175242.png)

![Screenshot](img/Pasted%20image%2020260424175326.png)

![Screenshot](img/Pasted%20image%2020260424175407.png)

![Screenshot](img/Pasted%20image%2020260424175514.png)

![Screenshot](img/Pasted%20image%2020260424175552.png)

I'll try my luck with one of the above, but for the regular user login instead.

Something a bit interesting there is that the browser doesn't send a POST to `login.php` but to `functions.php`. This seems to only have to do with how they set up the site though. That `login.php` uses JS instead of an HTML form submission.

After two attempts I got a response I liked:

![Screenshot](img/Pasted%20image%2020260424181358.png)

The response seems to suggest that my injection succeeded and that it's trying to redirect me to `dashboard.php?isadmin=false`. If I try to type my payload on the website itself though, I get a warning about using disallowed characters...

The endpoint `dashboard.php` doesn't seem to exist either, so I run a Feroxbuster to be sure. Might find something else good along the way.

Apart from a bunch of endpoints for phpmyadmin I found `/flags`, but I don't have permission to view it. No `dashboard.php` though...

But now, as if by a miracle, when I reloaded the page and clicked the Login menu item, I got redirected to `/dashboard.php` while logged in! (Okay, maybe it wasn't a miracle, just me forgetting a bit how Burp works :D)

It was my payload with `dev@injectics.thm'#` that worked. Correct email, and then the statement is closed with `'` and the rest is commented out with `#`

![Screenshot](img/Pasted%20image%2020260424182921.png)

I don't find a flag though, so the question is whether I'm logged in in the right place.

![Screenshot](img/Pasted%20image%2020260424183037.png)

Tried typing `dashboard.php?isadmin=true` (and false) but it makes no difference. Mysterious.
## mail.log
My good friend Onind00 made an exciting discovery that could potentially lead somewhere interesting. I had myself found a comment in the source code saying that mail is saved to `mail.log`. I thought this was a file/endpoint you couldn't reach from the outside, but no! His fuzzing (but not mine, boohoo) of endpoints had gotten a hit on exactly that. What's hiding there?

Oh oh oh.

![Screenshot](img/Pasted%20image%2020260424201422.png)

In other words. If (read: *when*) the database gets deleted, two new accounts with default credentials will be inserted according to the list. So they never lose access. Hehe. Safety first, right?

So a little SQLi that happens to delete the database would be fitting - then it's just a matter of logging in with the credentials above.

Note. I tested logging in with the credentials directly without SQLi - they don't seem to be active at the moment.

Then I believe in the following attack chain: Bypass the login for regular users with SQLi so I log in as `dev`, then SQLi in the form where you can edit the leaderboard. That injection deletes the database and the developer's function that automatically adds default accounts gets triggered. Then I should be able to access the first flag.
## The Leaderboard

Tried using my payload on the "superadmin account", that worked just as well. Now `admin` is welcomed, but there's no further functionality.

![Screenshot](img/Pasted%20image%2020260424212803.png)

I've had fun defacing all the stats, so everything's at 0. But I'm thinking maybe this is where there's a possibility of deleting the database entirely.

When I go in and edit an entry on the leaderboard, the following POST request is sent to `edit_leaderboard.php`

`rank=1&country=&gold=1&silver=2&bronze=3`

A bit exciting is that there's a variable called `country` that's empty. Maybe a possible entry point for SQLi?

Anyway, I copied the whole request I'd captured in Burp, saved it to a file and used it with sqlmap as follows:

`sqlmap -r request.txt --level=3 --risk=2 --dbs`

Unfortunately no successful result with that command. But I took the opportunity to do a run against the username field - where I already know it's possible to bypass without a password with SQLi. However I left the computer for a bit too long, so the machine ran out of time, but the last thing sqlmap had returned was this:

![Screenshot](img/Pasted%20image%2020260425005707.png)

So, probably MySQL is used (which was admittedly my guess from the very start), and possibly it's possible to delete the database from the login form? Trying more tomorrow.

![Screenshot](img/Pasted%20image%2020260426164749.png)
## New day, new machine

Ran sqlmap again and it can confirm that the username field is vulnerable to SQLi (which I admittedly already knew) and gave some more good info:
- Server OS: Ubuntu
- Stack: Apache with MySQL 5.0.12

![Screenshot](img/Pasted%20image%2020260425101525.png)

So I know you can run a `SLEEP(5)` and that the server waits with the response - so it should also be possible to delete all the databases.

Turned out you can't wildcard that way with MySQL. I need the name of a database or table.

However sqlmap can send a request to the current database without providing a name, so even though I hadn't managed to enumerate the table names, I could try deleting tables that usually tend to exist.

So I tried with `sqlmap -r req.txt --flush-session --sql-query="DROP TABLE users;"`, but got this response:

![Screenshot](img/Pasted%20image%2020260425111421.png)

Then it's probably as I thought from the start - the login form is the way in, but the leaderboard is probably the place to exploit to delete the database.
## The leaderboard again

Captured the POST request for the leaderboard and then ran `sqlmap -r leaderboard.txt --flush-session --level=3 --risk=3 --dbs`

Didn't find anything exciting, so increasing level to 5 and doing a new run.

That got a hit on the `gold` parameter!

![Screenshot](img/Pasted%20image%2020260425113306.png)

Back to Burp to see if I can drop the users table. Users should exist after all.

![Screenshot](img/Pasted%20image%2020260425113446.png)

In Burp it says `Error updating data`, but when I reload the page I see that the gold medals for the other countries are gone. Maybe it worked anyway?

![Screenshot](img/Pasted%20image%2020260425113623.png)

Can't log in. Maybe had the wrong syntax.

![Screenshot](img/Pasted%20image%2020260425114141.png)

Reloaded the page, and it looks like I succeeded! Imagine how much of a difference there can be between `'` and `;`

![Screenshot](img/Pasted%20image%2020260425114223.png)

Trying to log in to the admin panel now with `superadmin@injectics.thm:superSecurePasswd101` and:

<details>
  <summary><b>Click here to see the flag</b></summary>

  ![Screenshot](img/Pasted%20image%2020260425114341.png)
</details><br>

First flag secured!
## Flag two
The next flag should be found in the `flags` folder. The admin panel presents a new function - a page for updating your profile, and I'm convinced that's the way in.

![Screenshot](img/Pasted%20image%2020260425115136.png)

Send off an update to capture the POST request. Fed it into sqlmap and did a run. Now with level 5 right away.

Okay, level 5 was maybe a bit overkill because it takes an insanely long time. Ran 10 min on the first parameter without a hit, so skipping ahead to the next one.

Hehe, thought I'd check the source code while sqlmap was running. Was met with this :D

![Screenshot](img/Pasted%20image%2020260425182910.png)

So, it feels like the first and last name fields are at least vulnerable to something. Just need to land on exactly what.

It keeps chugging along. While it's working I test the `--os-shell` flag from the same entry point at the leaderboard - maybe I don't need to find a new entry point. This with `sqlmap -r leaderboard.txt -p gold --level=5 --risk=3 --os-shell --batch`

(Tried first without level and risk since I'd already scanned deeply, but that didn't work)

Getting a shell from the leaderboard doesn't seem to work.

![Screenshot](img/Pasted%20image%2020260425185639.png)

Let sqlmap run for a long time now, so time for manual tests.

Adding a `'` to each field becomes step 1.

No effect on first name

![Screenshot](img/Pasted%20image%2020260425202154.png)

Not on email either.

It occurred to me that the module had also covered template injection - no wonder sqlmap didn't find any vulnerabilities.

I tried sending `{{7*7}}` as first name and got 49 back!

![Screenshot](img/Pasted%20image%2020260425205438.png)
## Template injection

Time to figure out which framework is running then.

`{{7*'7'}}` also gave 49, so it could be Twig. (Had it been Jinja2 the output would have been `777777`.)

Now it's time to look in the module, because the syntax here gets really tricky...

Tried `{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}` but that gave:

![Screenshot](img/Pasted%20image%2020260425210535.png)

It wasn't the easiest to find what's allowed - it's pretty locked down...

I've gotten help from Claude and tested a whole bunch of different commands, but they're either *not allowed* or *unknown*. It really matters to find exactly the right approach.
## sstimap

Trying to install **sstimap** to see if it can both confirm which template engine is running (admittedly convinced it's Twig), and find the right exploit.

After a couple of attempts I landed on the following command:

`sstimap -u "http://$IP/update_profile.php" -m POST -d "email=superadmin%40injectics.thm&fname=*&lname=hej" --cookie "0me0adiqgbfmvtjfknl3gnk6lp" --engine Twig`

However sstimap said the form isn't vulnerable. Weird. Although not that weird really, since the rendered output isn't shown on the same page as the form. The form is on `/update_profile.php`, while the welcome message shows up on `/dashboard.php`. This is Second-Order SSTI, and after checking the help menu and searching online, sstimap doesn't seem to have any decent support for this. Unfortunately...

Found a feature request (with a suggestion for how it could be implemented) on github: https://github.com/vladko312/SSTImap/issues/12

`{{attribute(_self, 'getEnvironment')}}` still gave something exciting, but not that helpful if I understand it correctly.

![Screenshot](img/Pasted%20image%2020260426143614.png)

Can't find a way in - nothing is allowed!

![Screenshot](img/Pasted%20image%2020260426143739.png)

![Screenshot](img/Pasted%20image%2020260426143719.png)

![Screenshot](img/Pasted%20image%2020260426143814.png)

![Screenshot](img/Pasted%20image%2020260426143840.png)

However found a really good cheat sheet on [HackTricks](https://hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html) for checking which template engine is being used:

![Screenshot](img/Pasted%20image%2020260426144512.png)

![Screenshot](img/Pasted%20image%2020260426165028.png)
## The breakthrough

A small breakthrough with `{{['id','']|sort('passthru')}}` because it actually prints out which user we're running as:

![Screenshot](img/Pasted%20image%2020260426163236.png)

Swapped `id` for `ls` and got the following:

![Screenshot](img/Pasted%20image%2020260426163435.png)

`ls flags` (first ran `ls -la /flags` but it showed nothing, removed the slash afterward)

![Screenshot](img/Pasted%20image%2020260426163522.png)

`cat flags/5d8af1dc14503c7e4bdc8e51a3469f48.txt`

<details>
  <summary><b>Click here to see the flag</b></summary>

  ![Screenshot](img/Pasted%20image%2020260426163625.png)
</details>

Flag two secured!

I'm basically 100% sure I tested `passthru` yesterday, but surely I made the same mistake then that I happened to make at first today. I wrote `passthrough` because *through* is usually spelled that way and I've put a lot of energy and effort into learning that tricky spelling..! Well. Once again, a typo can be the only thing standing between you and the flag :D Happy hacking!
