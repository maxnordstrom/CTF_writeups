> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Have a Break

![Screenshot](img/Pasted%20image%2020260407165055.png)

https://tryhackme.com/room/haveabreak

![Screenshot](img/Pasted%20image%2020260407165129.png)

## Question 1

![Screenshot](img/Pasted%20image%2020260407165626.png)

We need to figure out which VPN service was used to send the anonymous email. I check the email's headers and look up the IP addresses I can find.

The first IP address leads to Google.

The second to 31173 Services AB, which is a VPN service, but I don't find a name that matches what they're looking for in the answer.

![Screenshot](img/Pasted%20image%2020260406152722.png)

Need to keep looking.

Found this one but it's a private address:

![Screenshot](img/Pasted%20image%2020260406152938.png)

Went back to the address that was actually a VPN service. Did a search on `whois.domaintools.com` (could just as well have run `whois` in the terminal, admittedly...) and saw that the company was registered at a Swedish address:

![Screenshot](img/Pasted%20image%2020260406153710.png)

How many Swedish VPN services are there with a 7-character name? Tried `Mullvad` and that was correct. Wonder how you could find information that actually confirms this, but oh well.

I kept searching to try to get it fully confirmed. Found a site called `ipinfo.io`, and if you created an account and went to `ipinfo.io/IP` you got info about which VPN service was used. Good site to remember!

![Screenshot](img/Pasted%20image%2020260407091435.png)

## Question 2

![Screenshot](img/Pasted%20image%2020260407091607.png)

We have the following image to work from:

![Screenshot](img/Pasted%20image%2020260407101117.png)

After many searches on google maps I combined an image search with `overnight parking` and got the following:

![Screenshot](img/Pasted%20image%2020260407100702.png)

Doesn't seem to be the right one either though, it's too close to Brno.

Let's look a bit more at the document we've been given describing the image:

![Screenshot](img/Pasted%20image%2020260407101246.png)

The image should have been taken near Hulín... Back to google maps. Although when I re-read it, it says the image was *seized* near Hulín - so the image doesn't necessarily have to have been taken there...
## Question 3

![Screenshot](img/Pasted%20image%2020260407101859.png)

Got an access log, taking a look at it.

We've also been given access to an email stating that someone noticed suspicious activity the night before departure.

![Screenshot](img/Pasted%20image%2020260407102119.png)

Here are the two last events in the log that could be suspicious...

![Screenshot](img/Pasted%20image%2020260407102257.png)

22:14:09 was the correct answer to that one.
## Question 4

![Screenshot](img/Pasted%20image%2020260407102351.png)

The sender of the email is `notmyname2847@gmail.com`

We've been given a staff list to work from:

![Screenshot](img/Pasted%20image%2020260407102540.png)

There's no connection between the email address and anything in the staff list.

We can see that the email was sent from an Android phone:

![Screenshot](img/Pasted%20image%2020260407102814.png)

But judging by the email, it's someone who saw suspicious activity in the internal systems, and someone who sees that kind of activity is probably someone in IT? Worth testing.

That wasn't right... You could brute force the whole thing, there aren't that many staff after all, but that's not really the right way to solve it.
## Question 5

![Screenshot](img/Pasted%20image%2020260407103202.png)

It should be the person who did the export of the route document. And it was!

## Question 6

![Screenshot](img/Pasted%20image%2020260407104203.png)

I thought it would be as simple as checking the staff list, but there are no names listed there.

In a communications report it says that an external email address requested access to a file, but was blocked. The address is `kraliknovak09@gmail.com`, but a google search on it doesn't give much. Nothing more than that over 280 searches have been made on the address in the last 7 days :D

Found another site for looking up email addresses, `epieos.com`

![Screenshot](img/Pasted%20image%2020260407130531.png)

Let's see if it lives up to its name.

After registering I did a search and got the following

![Screenshot](img/Pasted%20image%2020260407131248.png)

There we can see that the person in question was "busy" between 22 and 00 on the 26th of March

![Screenshot](img/Pasted%20image%2020260407131413.png)

And when I followed the link to Google Maps it seemed to point to a local guide

![Screenshot](img/Pasted%20image%2020260407131549.png)

The question now is whether this is the name of the leak, or the person who sent the anonymous email?

Switched tab to Reviews and found a rating for a certain gas station...

![Screenshot](img/Pasted%20image%2020260407131840.png)

The address of that gas station was the correct answer to **question 3**. Yey!

What about Radovan then, could that be the answer to the last question? Yep, that was the correct answer to **question 6.** Great!

Now on to question 4, which of the employees sent the anonymous email...

The email was also sent from a gmail address, specifically `notmyname2847@gmail.com`. Might be something interesting to find there.

Unfortunately that gave no hits

![Screenshot](img/Pasted%20image%2020260407134150.png)

Then it's a really tricky question...

I read through the attached pdf file again about what needs to be done.

![Screenshot](img/Pasted%20image%2020260407143119.png)

So, we've looked at a few headers in the attached email - maybe there's additional metadata that reveals an employee ID? I don't find any additional info that seems useful though...

Now we turn to the more obscure methods. Sent an email to the address in hope of a revealing auto-reply. No luck. Tried logging into the account. No luck, the account doesn't seem to exist...

![Screenshot](img/Pasted%20image%2020260407152254.png)

Now I took a guess where my theory is that it's the person who logs into the system right after the export who reacts to someone having been in at a notable time.

![Screenshot](img/Pasted%20image%2020260407162326.png)

So it's a Dispatch Operator who logs in, notices the mystery, and then emails the local newspaper.

I feel a bit cheated of the logic on this particular question, since in theory it could be any employee at all who logs into the systems after the export has taken place. Will be interesting to read more writeups on this one.

But, all flags found!
