> *This translation has been made by Claude, check out the original writeup [here](readme.md).*

# Cache Me Outside

*2026-06-08*

![Screenshot](img/Pasted%20image%2020260608093003.png)

https://tryhackme.com/room/cachemeoutside
## Intro

> Years after walking away from the scene, a retired hacker has left pieces of his identity scattered across the open internet.
>
> At first glance, it looks like nothing more than a leaked conversation screenshot. But buried in that image is the first thread of a much larger trail. Public profiles, forgotten details, and small mistakes begin to connect into something more deliberate.
>
> Someone wanted this person found.
## Task

> You are an OSINT investigator tasked with identifying the retired hacker and tracing the clues he left behind.
>
> Start with the conversation screenshot, follow his online presence, connect the exposed details, and use the final evidence to determine where the trail ends.
## Flag 1 (Full Name)

Followed the link found in the image:

![Screenshot](img/Pasted%20image%2020260608093046.png)

That led to a social media site (Komoot), where the name was shown.
## Flag 2 (Email)

On Komoot there was a link to github. I looked at the only repo and its only commit. Added `.patch` to the URL and could read out the email address.

<details>
  <summary><b>Click here to see the email</b></summary>

  ![Screenshot](img/Pasted%20image%2020260608093326.png)
</details>

## Flag 3 (Phone)

We searched for a long time without finding it - checked social media, google maps, calendar, epieos.com without success. Early in the process I'd floated the idea of sending an email to the email address in the hope of an autoreply containing the phone number, but it felt like a long shot. In the end **Tobzon** sent an email and got an instant autoreply :D
## Flag 4 (City)

Found an account for the user `jiml33t` on Threads. There was a picture there where a house had `irigatii.ro` on the facade. Searched google maps and found the head office. The city's name matched the answer.
## Flag 5 (Tram Station)

Looked for a French supermarket in the same city as above. Didn't find one though, so went with the same location as the head office. A few stops away the correct answer was found.
