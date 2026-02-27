# Confidential

https://tryhackme.com/room/confidential

![Screenshot](img/Pasted%20image%2020260226125713.png)

Man ska ha fått tag i en pdf-fil och i den ska det finnas en QR-kod vi vill ha. Men QR-koden är tydligen dold bakom några bilder, så vi måste försöka ta bort bilderna eller ta fram QR-koden på något annat vis.

![Screenshot](img/Pasted%20image%2020260226125845.png)

Rummet körs i split sqreen på THMs hemsida.

Här hittas en tjusig lite pdf.

![Screenshot](img/Pasted%20image%2020260226130118.png)

![Screenshot](img/Pasted%20image%2020260226130233.png)

Körde `pdfinfo` på filen och fick lite info

![Screenshot](img/Pasted%20image%2020260226131858.png)

Därefter `pdfimages` som bryter ut bilder ur en pdf, använde följande kommando: `pdfimages Repdf.pdf -all QR`. Fick då ut 3 bilder totalt.

![Screenshot](img/Pasted%20image%2020260226132022.png)

Visade sig att den första bilden innehåller originalet, alltså QR-koden utan maskning.

Scannade QR-koden med mobilen och vips så var flaggan i hamn.