> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# WARMUP: /files (Misc)

![Screenshot](img/Pasted%20image%2020251215232724.png)

A warmup challenge in the Misc category that leaves a lot to the imagination - there is no description. What you have to go on is the title, a picture of Inspector Gadget and the theme song for the same.

My teammates had already looked at the task and analyzed both the image and the audio file without finding anything directly interesting. One of the team mentioned that there was a sound in the music that stood out a bit and that it might contain something of value. I had a different approach.

### The challenge title

Since the challenge was called **/files** I figured there might be a directory on the server with that name, and that you could somehow find other interesting files there, for example a flag.

So I checked the inspector to see where the image was fetched from.

![Screenshot](img/Pasted%20image%2020251215231931.png)

In hindsight I can see that I already had the first half of the flag right there, but my focus was entirely set on looking for **/files**, and I found that there!

I opened the link in a new tab, which both downloaded the image and displayed it, although from my local directory, but it also opened another tab: `https://ctf.samurai.nu/files/Th1nk1ng_Out51/inspector-gadget.webp`

I tried navigating to `https://ctf.samurai.nu/files/` but that resulted in a 404.

Then I looked at the link and, aha, I saw something that could be a flag! I saved the string `Th1nk1ng_Out51`, went back to the task and did the same thing with the audio file, which happened to be at `/files/d3_Th3_Ch4ll3ng3_B0x/Inspector_Gadget_Theme.mp3`. There was the second half.

I wrapped both halves in the CTF's flag format and submitted. It was correct!

<details>
  <summary><b>Click to see the flag</b></summary>

  `O24{Th1nk1ng_Out51d3_Th3_Ch4ll3ng3_B0x}`
</details>
