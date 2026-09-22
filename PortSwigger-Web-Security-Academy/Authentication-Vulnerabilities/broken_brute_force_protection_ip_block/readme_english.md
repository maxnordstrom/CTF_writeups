> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Lab: Broken brute-force protection, IP block

![Screenshot](img/Pasted%20image%2020260318092805.png)

Step one is to try to find out what type of blocking is implemented and after how many failed logins. A central part covered by the module is that the IP logging can be reset as soon as you make a successful login. By regularly entering your own, valid credentials into a brute-force attack, you could therefore bypass the rate limit.

I understand in theory how to go about it, the question is how to do it in Burp. I start by testing a bit manually.

Made a login attempt with an obviously incorrect password to see if anything in the response headers reveals what's happening. However, there was nothing. Send it on to repeater and repeat.

After 4-5 failed login attempts this error message appeared:

![Screenshot](img/Pasted%20image%2020260318093454.png)

After making a successful login with my own credentials and then making a new attempt with `carlos`, the original error message appeared again:

![Screenshot](img/Pasted%20image%2020260318093635.png)

So - we need to configure an attack with Intruder where we throw in our own credentials at regular intervals to prevent my IP from being blocked.

After some searching I understood that the easiest way is to create two wordlists. I had hoped there would be some neat way to do this in Burp, but it's going to be a bit of manual work before we can kick off Intruder.

I created a list with the username `carlos` 100 times (as long as my password list) using `yes carlos | head -n 100 > usernames.txt`

I then inserted the valid username `wiener` on every fourth line using the command `sed '3~3 a\wiener' usernames.txt > usernames2.txt`

I then repeated the same process for my password list, so that the valid password `peter` appeared on every fourth line.

![Screenshot](img/Pasted%20image%2020260318100228.png)

![Screenshot](img/Pasted%20image%2020260318100329.png)

Time to run Intruder.

Set up a pitchfork attack that I thought would work

![Screenshot](img/Pasted%20image%2020260318101145.png)

But, even though the correct credentials are sent, we get the same error message that we've hit the rate limit...

Only on two occasions does the login succeed with my valid credentials:

![Screenshot](img/Pasted%20image%2020260318101442.png)

This feels a bit strange...

What seems most likely is that we're trying too many `carlos` attempts before a valid `wiener` comes along. Editing my wordlists so that my valid credentials come on every third line instead.

That seemingly made no difference...

Testing starting with my correct credentials instead. Changed the value of my payload positions so that value 0 is my valid credentials instead of just being a placeholder.

![Screenshot](img/Pasted%20image%2020260318102640.png)

After that, Intruder played out as I intended:

![Screenshot](img/Pasted%20image%2020260318102713.png)

Every third request, with my valid credentials, results in a 302 redirect, great. Time to look for a successful login for `carlos`

Sorted by status code and could quickly see the username we were looking for:

![Screenshot](img/Pasted%20image%2020260318102836.png)

The correct answer is therefore `carlos:jordan`

Successful login!

![Screenshot](img/Pasted%20image%2020260318102911.png)
