# Year of the Rabbit

https://tryhackme.com/room/yearoftherabbit

![[Pasted image 20260204084523.png]]

![[Pasted image 20260204084741.png]]

![[Pasted image 20260204084754.png]]

## Initial Recon

`nmap -sS -sV 10.81.152.190 -v`

![[Pasted image 20260204085206.png]]

Körde curl mot IP:t och det var Apache2 Default Page.

![[Pasted image 20260204090107.png]]

Körde en Feroxbuster med en enkel wordlist. Inget direkt av värde, blev bara RickRolled...

![[Pasted image 20260204090356.png]]

Testar att leta info genom anonym inloggning via ftp. Men det gick inte.

Kollade efter ledtrådar i CSS-filen och hittade en!

![[Pasted image 20260204091052.png]]

Gick till URL:en och fick följande alert:

![[Pasted image 20260204091156.png]]

När man klickar på OK kommer man till en rickroll video igen... Men kanske ligger något under alerten?

Vi curlade endpointen istället (för att undvika JS) och fick följande:

![[Pasted image 20260204091840.png]]

Curlade nästa

![[Pasted image 20260204092109.png]]

Sidan har flyttat och jag ser inte till vad i curl, kollar webbläsaren

Visade sig att jag gjort fel, skulle ha med snedstrecket på slutet. Nu funkade curl, och i webbläsaren såg det ut så här:

![[Pasted image 20260204092618.png]]

Laddade ner Hot_Babe.png och körde strings, där fanns lite ledtrådar:

![[Pasted image 20260204092813.png]]

Så nu har vi rätt användarnamn till FTP och ett antal lösenord att bygga en wordlist med. Dags att köra hydra mot FTP:n

`hydra -l ftpuser -P wordlist.txt ftp://$IP`

![[Pasted image 20260204094134.png]]

Loggade in via FTP och hittade följande, laddade ner den

![[Pasted image 20260204094601.png]]

Innehåller brainfuck

![[Pasted image 20260204094634.png]]

Gick till https://www.dcode.fr/brainfuck-language och fick ut följande:

![[Pasted image 20260204094744.png]]

`eli:DSpDiM1wAEwid`

Testar det på FTP:n eftersom det är enda inloggningsmöjligheten vi hittat. Inloggningen lyckades och vi hittade följande:

![[Pasted image 20260204095156.png]]

Alla mappar är tomma, men filen  `core` verkar intressant. Laddar ner den och det är en elf-fil.

![[Pasted image 20260204095241.png]]

Körde strings men hittar ingen uppenbar flagga - känns som att vi ska köra programmet för att kunna hoppa vidare på nåt vis. Dags att googla lite.

## User Flag

Onind00 testade att använda samma credentials för SSH, och det gick :D

När man ssh:ade in fick vi följande meddelande:

![[Pasted image 20260204125622.png]]

Valde att söka efter delar av den "hemliga" strängen

`find / -type d -name "*s3cr*" 2>/dev/null`

Och fick följande träff:

![[Pasted image 20260204130325.png]]

![[Pasted image 20260204130400.png]]

Hittade det hemliga meddelandet, och vi kan säkert använda det för SSH. `gwendoline:MniVCQVhQHUNI`

![[Pasted image 20260204130441.png]]

Och vi hittade user flag

![[Pasted image 20260204130558.png]]

## Root Flag

Körde `sudo -l` för att se om gwendoline har några högre rättigheter, och det hade användaren.

![[Pasted image 20260204131418.png]]

Gick direkt till GTFO-bins och kollade vad för kul man kan göra med **vi**

Kunde inte läsa några filer i `/root`

Kunde spawna ett shell, men inte ett root shell.

Vi får ju köra vi som *alla* användare, men inte root.

Efter ett gäng sökningar visade sig att det finns en CVE som heter CVE-2019-14287

![[Pasted image 20260204133935.png]]

![[Pasted image 20260204133922.png]]

Jag fick det dock inte helt att funka när jag körde på det sättet som bloggen visade. Kollade NIST istället:

![[Pasted image 20260204143743.png]]

Här är lite intressant, för om jag gör så som jag läst på flera sidor nu står det endast att jag inte får köra `/usr/bin/vi` som `#-1`.

![[Pasted image 20260204144412.png]]

Men om jag försöker köra `vi` mot just filen som är specificerad så öppnas filen med user-flaggan. Alltså när jag kör 
`sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt`

Det visade sig att när jag väl har öppnat **vi** (och angett `-u#-1`) så ska jag köra `:!/bin/bash` för att spawna ett root shell.

![[Pasted image 20260204145833.png]]

Och flaggan hittades i root-katalogen

![[Pasted image 20260204145915.png]]

Jag testade att justera mitt kommande baserat på vad jag läst på GTFO-bins

![[Pasted image 20260204150750.png]]

Men varje gång jag försöker lägga till `-c ':!/bin/bash'` på slutet så behöver jag uppge lösenord och får felmeddelande om att jag inte har behörighet. Så det verkar som att man måste göra det i flera steg - först öppna filen med sudo, sedan spawna ett shell. Istället för att skriva `:!/bin/bash` kan man även skriva `:shell`