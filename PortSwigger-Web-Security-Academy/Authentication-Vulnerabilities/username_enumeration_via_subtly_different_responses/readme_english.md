> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Username enumeration via subtly different responses

![Screenshot](img/Pasted%20image%2020260211115039.png)

Since the task wants me to look for small differences in the response, I'll use the same approach as before - capture a login attempt, send it to Intruder. Run brute force on the usernames (based on the given list) and hope I find something that says the password is wrong (instead of the username being wrong). Then brute force the passwords to find the right one.

The error message I get for every request in Intruder is `Invalid username or password`. So I can't determine anything based on the error message.

The response time does differ, though. The users `admin` and `carlos` have a noticeably shorter response time - maybe a sign that they were found quickly in the username list?

The other usernames have a longer time - possibly a sign that the server has had to go through the *entire* list before finally concluding that the username doesn't exist.

The user `af` also has a somewhat shorter response time, but not as short. Same with `arlington`.

My social engineering skills also tell me that `carlos` should exist, since that user has appeared in previous labs.

Testing a **cluster bomb attack** with `admin` and `carlos` as usernames, and the password list for, well, the password.

Hm, neither of the two seems to have resulted in a successful login. Some response times are admittedly significantly shorter than the average, and one is twice the average, but all get the error message `Invalid username or password`. All also have status code 200 - a successful login could possibly have some redirect code.

Testing the same thing but with the usernames `af` and `arlington`.

Suspecting I won't get much further with these either.

## Starting from scratch

Reading the instructions again.

> To solve the lab, enumerate a valid username, brute-force this user's password, then access their account page.

I remember that a tip was given earlier in the module - to enter a very long password. So once you get a hit on a valid username, it will take the server time to process the long password. Let's try that instead.

![Screenshot](img/Pasted%20image%2020260211201749.png)

Hm, it's still `af` that has the longest time at 192, followed by `apache` at 166. Testing `af`.

No luck. I try again with a very, very long password instead...

![Screenshot](img/Pasted%20image%2020260211202905.png)

With the long password, `administrator` has the shortest response time, while the following four have the longest:

![Screenshot](img/Pasted%20image%2020260211203408.png)

I first test `at`. Then a cluster bomb with all of them.

No successful result with `at` either. New attempt tomorrow...

## New day

I noticed that a number of responses contain an empty html comment, roughly a quarter of the responses. I recall that they mentioned exactly that in the module, that some responses can differ slightly. That they're very similar to a normal error message, but the fact that they differ somewhere indicates that a different page is being loaded, and it's presumably loaded for a reason. Maybe because the username is valid.

I'll pick out all the usernames that return the html comment and try to brute force them.

The annoying part is that 57 usernames get the empty html comment, 44 don't have it. That's too large a number for an efficient brute force...

Okay, now I had a flash of insight. Finally. I submitted a username that obviously doesn't exist and looked at the response - it contains the empty html comment. So, those with an empty html comment in the response are usernames that don't exist. That's my theory.

But what the heck... Now when I do a few spot checks it seems completely random... The ones that got the comment before don't get it now and vice versa.

I've now tested sending a request to Repeater, and it's really random whether the response contains the empty html comment or not... So that's not what distinguishes a request with a valid username. Definitely going to sleep on this one...

## Another new day

Today, for some reason, the response time for the user `test` stands out quite a bit:

![Screenshot](img/Pasted%20image%2020260219111723.png)

The response time in second place is 99.

After going through the ideas presented in the module, I went hunting for a typo, and I took a chance that there would be a typo in the error message, and there was! For the user `ae`, a period is missing at the end of the error message:

![Screenshot](img/Pasted%20image%2020260219135357.png)

All other responses contain the error message `Invalid username or password.` - i.e. with a period at the end.

I discovered this through manual work, by clicking through all the responses and scrolling down to the error message. I couldn't find any other, smoother way to do this in Burp Community Edition.

The machine, however, managed to shut down before I could test the passwords, and on restart the username `ae` was no longer valid - they seem to rotate valid users. But the theory about the missing period still held, because I saw the same pattern for the user `adkit` after the restart. Once I found the valid username, it was just a matter of running a password brute force and logging in.

![Screenshot](img/Pasted%20image%2020260219140942.png)
