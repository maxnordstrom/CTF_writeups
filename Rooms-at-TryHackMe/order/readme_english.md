> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Order

https://tryhackme.com/room/hfb1order

![Screenshot](img/Pasted%20image%2020260227145334.png)

![Screenshot](img/Pasted%20image%2020260227145347.png)

Ran an XOR brute force in CyberChef with the known plaintext, but that didn't give much.

Turned out I'd only pasted in half the message into CyberChef - I thought it was two different ones. Now I paste in the whole thing and get a bit more data to work with at least.

We do have part of the plaintext, but when we use it as the key in an XOR operation it's mostly garbage characters.

After a bit of searching it turned out I needed to decode the encrypted text from hex before running XOR.

I threw in a **From Hex** before the XOR operation and then we get what appears to be the key:

![Screenshot](img/Pasted%20image%2020260227152857.png)

Now I enter SNEAKY as the key instead and get:

<details>
  <summary><b>Click here to see the flag</b></summary>

  ![Screenshot](img/Pasted%20image%2020260227152933.png)
</details><br>

So, to summarize how XOR works:

- plaintext XOR key = ciphertext (this is how you normally encrypt)
- ciphertext XOR plaintext = key
- key XOR ciphertext = plaintext

And that's what we did above. We took the plaintext (ORDER:) and XOR'd it with the ciphertext and got the key (SNEAKY). Then we ran key XOR ciphertext and got the plaintext. And the flag!
