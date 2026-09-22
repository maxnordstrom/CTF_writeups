> *This translation has been made by Claude, check out the original writeup [here](README.md).*

![Screenshot](img/Pasted%20image%2020260130201133.png)

My idea is to send the POST request for uploading shell.php to Repeater so I can modify the filename and experiment.

Okay, I can only upload jpg and png

![Screenshot](img/Pasted%20image%2020260130201427.png)

With the request in Repeater I start by changing the header `Content-Type:` to `image/png` (instead of application/x-php)

Testing naming the file `shell.php.png`. Got this response:

![Screenshot](img/Pasted%20image%2020260130201826.png)

And it seems to have been uploaded as my avatar:

![Screenshot](img/Pasted%20image%2020260130201902.png)

What happens if I try to access it? Then I just got to the image:

![Screenshot](img/Pasted%20image%2020260130201954.png)

What if I run a GET request in Burp instead? Then I saw the file's content:

![Screenshot](img/Pasted%20image%2020260130202055.png)

Maybe I should leave `Content-Type` as `application` after all? Testing with that. It could be uploaded:

![Screenshot](img/Pasted%20image%2020260130202209.png)

In the browser I saw the same thing as before, and also in Burp. In the response in Burp I could see that Content-Type had been changed to image/png, so that must have happened server-side.

Testing running a null byte before .png. That is, `shell3.php%00.png`. Then I got this response

![Screenshot](img/Pasted%20image%2020260130202706.png)

Looks like it uploaded a pure php file this time. If I run a GET request in Burp? Then I got the flag.

![Screenshot](img/Pasted%20image%2020260130202755.png)
