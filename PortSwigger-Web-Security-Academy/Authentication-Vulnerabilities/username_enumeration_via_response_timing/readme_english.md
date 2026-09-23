> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Username enumeration via response timing

![Screenshot](img/Pasted%20image%2020260219141542.png)

Since I have working credentials, I plan to try logging in with the valid username but with an incorrect password to see how long it takes for the server to respond. Then with an obviously incorrect username to see if the server takes longer or shorter to respond.

Once I have a sense of the response times, I run a username enumeration and look for a response time that matches the time I got for the username `wiener`.

Here are my attempts with the user `wiener`, both with invalid and the valid password:

![Screenshot](img/Pasted%20image%2020260311151950.png)

Now I use the same list but with a username that's invalid, I landed on the username `TheKingOfKingsAndQueensAndPrinces`. Here's the result:

![Screenshot](img/Pasted%20image%2020260311152221.png)

No *aha moment*, so I guess I'll have to burn through the whole list of usernames and see if anything stands out.

On a first run I had `password` as the password, but when I go back and reread the text from the module it says this:

![Screenshot](img/Pasted%20image%2020260311152741.png)

So I can run a brute force with the usernames, and enter an unnecessarily long password as the payload to see if it gives a longer response time.

With a short password the response times are fairly evenly spread between 55 and 217, nothing that stands out too much.

Testing with a long one instead

![Screenshot](img/Pasted%20image%2020260311155351.png)

Then the majority of responses are around 90, while a couple climb a bit above 100. I'll test a Cluster bomb attack with `administrator` and `application` together with the list of valid passwords.

![Screenshot](img/Pasted%20image%2020260311160155.png)

Now I looked a bit closer at the response and noticed this fun little thing:

![Screenshot](img/Pasted%20image%2020260311160622.png)

Feels like it goes a bit outside the theme of the lab, but I probably need to find a way to avoid the rate limit. Maybe that's what's under the hint? :D I haven't checked, but I'll give it a try first.

I added the header `X-Forwarded-For:` (from the foundational course Bypass Rate Limit 101, haha) and set up a Pitchfork attack to bypass the rate limit. Payload 1 generated a number between 0 and 255 (to simulate a valid IP address), while Payload 2 went through the usernames.

![Screenshot](img/Pasted%20image%2020260311162723.png)

I suspect I found the right username ;)

![Screenshot](img/Pasted%20image%2020260311162816.png)

Since the response time is so much longer, you can suspect that it's the only username the server tested the password against.

Time to run a brute force on the username `ftp`. And I keep my X-Forwarded-For header.

It resulted in a Redirect 302 with the password `harley`

![Screenshot](img/Pasted%20image%2020260311163135.png)

And I managed to log in with `ftp:harley`

![Screenshot](img/Pasted%20image%2020260311163214.png)

Haha, I checked the hint now, and it was as I thought :D

![Screenshot](img/Pasted%20image%2020260311163858.png)
