# Stolen Mount

*2026-06-08*

![Screenshot](img/Pasted%20image%2020260608122553.png)

https://tryhackme.com/room/hfb1stolenmount
## Intro

> An intruder has infiltrated our network and targeted the NFS server where the backup files are stored. A classified secret was accessed and stolen. The only trace left behind is a packet capture (PCAP) file recorded during the incident. Your mission, should you accept it, is to discover the contents of the stolen data.

![Screenshot](img/Pasted%20image%2020260608122814.png)
## Recon

Öppnar pcap-filen i Wireshark och kollar protocol hiearchy, tittar närmare på NFS-trafiken

Följer stream.

Ser fragment av en lösenordshash, en zip-fil och en png-fil.

![Screenshot](img/Screenshot%20from%202026-06-08%2011-19-43.png)

md5-hashen var `avengers` enligt crackstation. Så om vi kan återskapa zip-filen är det väl bara att läsa innehållet?

Fixar så jag kan kolla på filen med Networkminer - det programmet kan extrahera zip-filer per automatik. 

Fick det dock inte att lira med NetworkMiner, så det blev att tänka om.
## Tshark

Fanns heller inget smidigt sätt att fixa detta i Wireshark, så för att dumpa den råa NFS-datan körde jag följande kommando med **tshark**.

```bash
tshark -r challenge.pcapng \
-Y "tcp.port == 2049 and tcp.len > 0" \
-T fields -e tcp.payload \
> | tr -d ':\n' | xxd -r -p > raw.bin
```

Därefter `binwalk -e raw.bin`.

![Screenshot](img/Pasted%20image%2020260608122239.png)

Katalogen `_raw.bin.extracted` skapades och där fanns zip-filen. Öppnade den med `unzip 8E64.zip` och angav lösenordet `avengers`.

![Screenshot](img/Pasted%20image%2020260608122353.png)

`secrets.png` innehöll en QR-kod som jag läste med `zbarimg` och vips så var flaggan i hamn!

<details>
  <summary><b>Klicka här för att se flaggan</b></summary>

  ![Screenshot](img/Screenshot%20from%202026-06-08%2012-17-27.png)
</details>