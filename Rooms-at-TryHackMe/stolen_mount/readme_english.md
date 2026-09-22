> *This translation has been made by Claude, check out the original writeup [here](readme.md).*

# Stolen Mount

*2026-06-08*

![Screenshot](img/Pasted%20image%2020260608122553.png)

https://tryhackme.com/room/hfb1stolenmount
## Intro

> An intruder has infiltrated our network and targeted the NFS server where the backup files are stored. A classified secret was accessed and stolen. The only trace left behind is a packet capture (PCAP) file recorded during the incident. Your mission, should you accept it, is to discover the contents of the stolen data.

![Screenshot](img/Pasted%20image%2020260608122814.png)
## Recon

Opening the pcap file in Wireshark and checking the protocol hierarchy, taking a closer look at the NFS traffic

Following the stream.

Seeing fragments of a password hash, a zip file, and a png file.

![Screenshot](img/Screenshot%20from%202026-06-08%2011-19-43.png)

The md5 hash was `avengers` according to crackstation. So if we can recreate the zip file, we should just be able to read the contents?

Setting things up so I can look at the file with NetworkMiner - that program can extract zip files automatically.

Couldn't get it to work with NetworkMiner though, so had to think of something else.
## Tshark

There was also no smooth way to fix this in Wireshark, so to dump the raw NFS data I ran the following command with **tshark**.

```bash
tshark -r challenge.pcapng \
-Y "tcp.port == 2049 and tcp.len > 0" \
-T fields -e tcp.payload \
> | tr -d ':\n' | xxd -r -p > raw.bin
```

Then `binwalk -e raw.bin`.

![Screenshot](img/Pasted%20image%2020260608122239.png)

The directory `_raw.bin.extracted` was created and the zip file was there. Opened it with `unzip 8E64.zip` and entered the password `avengers`.

![Screenshot](img/Pasted%20image%2020260608122353.png)

`secrets.png` contained a QR code which I read with `zbarimg`, and just like that the flag was secured!

<details>
  <summary><b>Click here to see the flag</b></summary>

  ![Screenshot](img/Screenshot%20from%202026-06-08%2012-17-27.png)
</details>
