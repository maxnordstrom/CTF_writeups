# ⚠️ Writeup not finished, work in progress...

# Challenge 3

## Introduction

This was a really tricky one, it took me several sessions to get it right. Not because it was overwhelmingly difficult or presented a task that totally differed from what the tutorials covered, but because of some kind of encoding working in a way I didn't expect. Let me show you how.

As I said, it took me several sessions, so I'll try to be as honest as I can remembering how it all went down. What I do remember for certain is that my goal was to retrieve the flag found in `/etc/flag3`.



## h2
In one of the tutorials I had to change the request (http-request?) from GET to POST, so I figured that could be the case here too. I asked a friend (yes, a real human friend, it's not only AI you know 😅) who told me a bit about how GET and POST differ, and that depending on how it's coded it can either be really easy to change or a real hassle. 

Lucky for me Try Hack Me went easy on me, and changing the method was as easy as having a look in the inspector and changing it.

IMG GET_to_POST_inspector