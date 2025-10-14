# CTF Collection Vol 1

<img src="img/hero.jpg" style="width:300px"></img>

> 🐱‍💻 Try out the room @ [tryhackme.com/room/ctfcollectionvol1](https://tryhackme.com/room/ctfcollectionvol1)

## 📋 Introduction

Denna writeup var egentligen tänkt som en enkel minnesanteckning för min egen del, men jag blev taggad på att snygga till den och publicera den här. Detta rum bestod av 20 CTF-utmaningar, och nu i efterhand så ser jag min writeup lite som en roadmap till hur man kan ta sig an de allra mest klassiska typerna av CTF. Jag hoppas att den kan komma till nytta för fler än mig själv 🙂 Jag tog mig an rummet tillsammans med [onind00](https://tryhackme.com/p/onind00) och vi hade en riktigt trevlig förmiddag.

## Task 1 - Author note

En välkomnande intro-text. Ingen flagga här inte.

## Task 2 - What does the base said? 💬

Uppgiften gick ut på att decode:a base64. I Linux körde vi `echo 'the_base64_string' | base64 -d`

## Task 3 - Meta meta

Vi använde `exiftool` för att läsa metadata

## Task 4 - Mon, are we going to be okay?

Vi använde `steghide` för att både se att filen innehöll ett hemligt meddelande, och samma verktyg för att extrahera meddelandet (för det krävdes inget lösenord, vi tryckte bara på enter).

## Task 5 - Erm... Magick 🔮

Vit text på vit bakgrund, vi hittade flaggan direkt

## Task 6 - QRrrr

Vi presenterades med en QR-kod som vi skannade och fick flaggan. Jag använde mobilen, men kompis använde Linux och `zbarimg -q --raw QR_1577976698747.png` 

## Task 7 - Reverse it or read it?

Filen har ändelsen `.hello` och verkar vara en binär fil på nåt vis. När jag öppnade den i notepad så ser jag en blandning av mystiska tecken och ASCII. När jag scrollade ner en bit hittade jag flaggan i klartext.

## Task 8 - Another decoding stuff

Vi presenterades med en sträng. Det var ingen uppenbar hash så vi skickade in den i CyberChef. Strängen var encode:ad med base58

## Task 9 - Left or right

Ytterligare en sträng. Den var krypterad med ROT13. Körde en brute force i CyberChef och fick ut flaggan. Det var amount 7 som gällde.

## Task 10 - Make a comment

Flaggan fanns i en paragraf som var dold med `style=display:none;`

## Task 11 - Can you fix it? 🛠️

Vi laddade ner en trasig png-fil. Läste filen med `xxd` och kunde se att de första bytesen var fel. En giltig png ska inledas med `89 50 4E 47` så jag körde `hexedit` och justerade, sparade och öppnade filen. Flaggan fanns mitt i bilden.

## Task 12 - Read it 👓

Vi skulle hitta en flagga någonstans på Tryhackme:s sociala medier, närmare bestämt Reddit. Inne på Reddit sökte vi på `r/tryhackme` och användaren som skapat rummet. Vi fick träff på en tråd och där hittade vi flaggan.

## Task 13 - Spin my head

Vi presenterades med en relativt obskyr sträng:

`++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>>++++++++++++++.------------.+++++.>+++++++++++++++++++++++.<<++++++++++++++++++.>>-------------------.---------.++++++++++++++.++++++++++++.<++++++++++++++++++.+++++++++.<+++.+.>----.>++++.`

Min kompis presenterade mig för hemsidan https://www.geocachingtoolbox.com/ och där hittade jag att strängen antagligen är ett program skrivet i `brainfuck`

Efter att ha skrivit in strängen och kört programmet på geocachingtoolbox fick vi flaggan.

## Task 14 - An exclusive!

Vi presenterades med strängarna

`S1: 44585d6b2368737c65252166234f20626d` och 

`S2: 1010101010101010101010101010101010`

`S1` är en hexsträng och `S2` binär. Vi gissade att vi skulle köra XOR på nåt vis. I CyberChef skrev vi in den första strängen, konverterade från hex, sedan körde vi XOR och använde den andra strängen som key. Då fick vi flaggan

## Task 15 - Binary walk

Vi presenterades med en jpg-fil, och uppgiften hette **Binary Walk**. Vi laddade ner bilden och körde `binwalk` och såg att filen innehöll en annan fil som hette `hello_there.txt`. Så vi körde `binwalk -e filename.jpg` och extraherade textfilen som innehöll flaggan.

Vi upptäckte även att vi kunde ladda upp bilden till CyberChef och ta **Scan for embedded files** eller **Extract files**.

## Task 16 - Darkness

Vi presenterades med en svart bild. Vi laddade ner bilden och öppnade den med `stegsolve.jar` eftersom vi misstänkte att det skulle vara en klassisk steganografi-uppgift. Hittade flaggan direkt.

## Task 17 - A sounding QR 🎶

En QR-kod som ledde till SoundCloud där vi kunde lyssna på på en röst som läste upp en text ganska snabbt. Jag spelade in ljudet med min mobil, sen öppnade jag ljudfilen i **Amazing Slow Downer** så att jag kunde höra vad flaggan var.

## Task 18 - Dig up the past ⛏️

Vi skulle kolla på hemsidan https://www.embeddedhacker.com/ vid en viss tidpunkt. Hoppade till Wayback Machine och skrev in den angivna tiden och hittade flaggan. 

## Task 19 - Uncrackable! 💥

Presenterades med en krypterad text och i instruktionen stod det att nyckeln var borttappad. Min tanke gick till Vigeneré-chiffret då det krävs en nyckel för att använda. Gick till CyberChef för att titta närmare. Det var en alldeles för liten sample text för att kunna knäcka Vigeneré, men vi testade med enkla nycklar (typ samma som svaga lösenord). Det var inte `key`, men det var `thm`.

## Task 20 - Small bases

Presenterades med en lång sträng med siffror, och baserat på uppgiftens namn borde det handla om att använda small bases på något sätt. Utan framgång med CyberChef använde jag AI för att skapa ett brute force-script i Python.

```json
s = "1202212101120"  # your string
for base in range(2, 37):
    try:
        val = int(s, base)
        out = val.to_bytes((val.bit_length()+7)//8, 'big')
        if all(32 <= b < 127 for b in out):
            print(base, out)
    except:
        continue
```

När jag matade den med den aktuella strängen och körde så fick jag flaggan.

## Task 21 Read the packet 📦

Sista rummet bestod av en pcap-fil. Jag öppnade den med Wireshark och sorterade på protokoll så kunde jag snabbt se att det fanns en intressant HTTP GET request. Jag högerklickade och tog Follow > HTTP Stream och kunde läsa flaggan.