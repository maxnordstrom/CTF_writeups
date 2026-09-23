> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# Confidential

https://tryhackme.com/room/confidential

![Screenshot](img/Pasted%20image%2020260226125713.png)

We're supposed to have gotten hold of a pdf file, and in it there's supposed to be a QR code we want. But the QR code is apparently hidden behind some images, so we need to try to remove the images or extract the QR code some other way.

![Screenshot](img/Pasted%20image%2020260226125845.png)

The room runs in split screen on THM's website.

Here we find a fancy little pdf.

![Screenshot](img/Pasted%20image%2020260226130118.png)

![Screenshot](img/Pasted%20image%2020260226130233.png)

Ran `pdfinfo` on the file and got some info

![Screenshot](img/Pasted%20image%2020260226131858.png)

Then `pdfimages`, which extracts images from a pdf, using the following command: `pdfimages Repdf.pdf -all QR`. That gave 3 images in total.

![Screenshot](img/Pasted%20image%2020260226132022.png)

It turned out the first image contained the original, i.e. the QR code without masking.

Scanned the QR code with my phone and just like that the flag was in the bag.
