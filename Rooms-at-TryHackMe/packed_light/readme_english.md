> *This translation has been made by Claude, check out the original writeup [here](readme.md).*

# Packed Light

![Screenshot](img/Pasted%20image%2020260811114744.png)

https://tryhackme.com/room/hh-packedlight-02e5330c
## Intro
![Screenshot](img/Pasted%20image%2020260811114828.png)

![Screenshot](img/Pasted%20image%2020260811114841.png)

Let's open the pcap file in Wireshark and get going!
## Initial analysis

1348 packets, so a relatively small capture.

The largest share appears to be TCP. On closer inspection it's SYN and ACK handshakes.

![Screenshot](img/Pasted%20image%2020260811115208.png)

When I skim through the HTTP packets, I see something interesting: `H0t3lSt@ff0Nly`, `K3epS3cr3t!` and `xor`. I suspect something has been run through an XOR function before exfiltration. Let's follow the stream.

![Screenshot](img/Pasted%20image%2020260811120305.png)

If I understand it correctly, it looks like the data is encrypted with XOR, encoded to base64, then to utf-8, and finally stored in the `Cookie` header. Worth digging up one or more of these cookies so I can get hold of some data to decrypt.

![Screenshot](img/Pasted%20image%2020260811121154.png)

Four characters? Hm... But every HTTP packet contains the header - time to piece it together then.

A command in tshark that prints the data I want to the terminal, a little sanity check.

![Screenshot](img/Pasted%20image%2020260811164359.png)

Each piece of data in the Cookie header is the output of a single keystroke. The reason they all end with `==` is that base64 always outputs groups of 4 characters, where `=` is padding.

So I first want to decode each group, individually, from base64 to get the encrypted text. After that I can run XOR with the key defined in `getKey()`

I've put together a file that only has my data of interest (yes, I manually wiped out the other stuff beforehand :D )

![Screenshot](img/Pasted%20image%2020260811165852.png)

A small script gave me this byte array

![Screenshot](img/Pasted%20image%2020260811170452.png)

I solved the rest in CyberChef. First From Hex, then XOR with the key. However, the result wasn't what I expected...

![Screenshot](img/Pasted%20image%2020260811171546.png)

The solution?

Since the XOR operation runs *every time* for *every key*, the whole key doesn't get used - only the first character, i.e. `H`. Change the key in the XOR?


Voilà, now I got the flag :)

<details>
  <summary><b>Click here to see it</b></summary>

  ![Screenshot](img/Pasted%20image%2020260811171709.png)
</details>
