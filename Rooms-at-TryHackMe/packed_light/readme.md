# Packed Light

![Screenshot](img/Pasted%20image%2020260811114744.png)

https://tryhackme.com/room/hh-packedlight-02e5330c
## Intro
![Screenshot](img/Pasted%20image%2020260811114828.png)

![Screenshot](img/Pasted%20image%2020260811114841.png)

Låtom oss öppna pcap-filen i Wireshark och köra!
## Initial analysis

1348 paket, en relativt liten capture alltså.

Största andelen ser ut att vara TCP. Vid en närmare titt är det SYN- och ACK-handshakes.

![Screenshot](img/Pasted%20image%2020260811115208.png)

När jag skummar igenom HTTP-paketen ser jag nåt intressant: `H0t3lSt@ff0Nly`, `K3epS3cr3t!` och `xor`. Misstänker att nåt har körts genom en XOR-funktion innan exfiltration. Låt oss följa strömmen.

![Screenshot](img/Pasted%20image%2020260811120305.png)

Om jag förstår det rätt så ser det ut som att datan krypteras med XOR, encodas till base64, därefter till utf-8 för att sedan sparas i headern `Cookie`. Värt att leta upp en eller flera såna cookies så jag får tag i nån data att dekryptera.

![Screenshot](img/Pasted%20image%2020260811121154.png)

Fyra tecken? Hm.. Men varje HTTP-paket innehåller headern - dags att pussla ihop då.

Ett kommando i tshark som som skriver ut datan jag vill ha till terminalen, en liten sanity check.

![Screenshot](img/Pasted%20image%2020260811164359.png)

Varje data i cookie-headern är outputen från en tangenttryckning. Anledningen till att de alla slutar med `==` är för att base64 alltid skriver ut en grupp på 4 tecken, där `=` är padding.

Jag vill alltså först decoda varje grupp, var för sig, från base64 för att få fram den krypterade texten. Därefter kan jag köra XOR med nyckeln som definieras i `getKey()`

Jag har fixat en fil som endast har mina intressanta data (ja, jag suddade ut sakerna innan manuellt :D )

![Screenshot](img/Pasted%20image%2020260811165852.png)

Ett litet script gav mig denna bytearray

![Screenshot](img/Pasted%20image%2020260811170452.png)

Resten löste jag i CyberChef. Först från Hex, sedan XOR med nyckeln. Dock blev resultatet inte vad jag förväntat mig...

![Screenshot](img/Pasted%20image%2020260811171546.png)

Lösningen?

Eftersom XOR-operationen kör *varje gång* för *varje tangent* så hinner inte hela nyckeln användas utan endast det första tecknet, alltså `H`. Ändra nyckeln i XOR?


Voilà, nu fick jag flaggan :)

<details>
  <summary><b>Klicka här för att se den</b></summary>

  ![Screenshot](img/Pasted%20image%2020260811171709.png)
</details>