# Passwords - A Cracking Christmas
### Advent of Cyber 2025, Day 9

https://tryhackme.com/room/attacks-on-ecrypted-files-aoc2025-asdfghj123

![Screenshot](img/Pasted%20image%2020251210165433.png)

## Intro
Rummet handlar om att knäcka lösenord och man får lite grundläggande info om hur det kan gå till. Man får förklaringen om att det sällan handlar om att "knäcka" lösenorden, utan snarare att gissa rätt med hjälp av olika ordlistor. Att köra en full brute force och testa *alla* tänkbara kombinationer är möjligt med rätt datorkraft och tid, men det är sällan lönsamt för the average pentester.

Uppgifterna i fråga handlar om att knäcka lösenorden till två lösenordsskyddade filer - en pdf och en zip. Jag kommer beskriva i korthet hur jag gick till väga, men det var inte de uppgifterna som var orsaken att jag tog mig till detta rum just idag. I slutet av uppgiften får man en hint om att man på denna maskinen kan hitta nyckeln för att låsa upp Sidequest 2!

Jag kommer beskriva i korthet hur jag gick till väga för att lösa uppgifterna och sedan presentera mer djupgående hur jag (och mina vänner) gick till väga för att hitta nyckeln till Sidequest 2.

## Uppgifter
Uppgifterna gick ut på att knäcka lösenorden till en pdf- och en zip-fil.

### PDF
Pdf-filen hette **flag.pdf** och jag använde programmet **pdfcrack**. Detta genom att köra kommandot `pdfcrack -f flag.pdf -w /usr/share/wordlists/rockyou.txt`
Programmet hittade lösenordet direkt och skrev ut det i terminalen.

<details>
  <summary><b>Klicka här för att se lösenordet</b></summary>

  `naughtylist`
</details><br>

Eftersom jag använde ssh för att komma åt servern kunde jag inte öppna pdf-filen på vanligt vis, utan fick konvertera den till text med programmet **pdftotext**. Detta genom kommandot `pdftotext -opw secretPassword flag.pdf` vilket istället gav mig filen **flag.txt**. `cat flag.txt` gav mig flaggan!

<details>
  <summary><b>Klicka här för att se flaggan</b></summary>

  `THM{Cr4ck1ng_PDFs_1s_34$y}`
</details>

### ZIP
Zip-filen hette **flag.zip** och jag använde John the Ripper för att knäcka lösenordet. För att john ska kunna arbeta med filen måste man först köra `zip2john` för att få en anpassad lösenords-hash. Detta genom `zip2john flag.zip > zip.hash`. Det gav mig filen **zip.hash** som john förstår. Därefter körde jag  `john --wordlist=/usr/share/wordlists/rockyou.txt zip.hash` varpå john hittade rätt lösenord och skrev ut i terminalen.

<details>
  <summary><b>Klicka här för att se lösenordet</b></summary>

  `winter4ever`
</details><br>

Jag provade att unzippa med **unzip** och **gzip**, menjag hittade inga alternativ för att lägga till ett lösenord. Det gick däremot med **7-Zip**. Detta genom kommandot `7z e flag.zip`. Jag fick därefter en prompt om hur jag ville spara den extraherade filen - jag valde att den inte skulle skriva över filen **flag.txt** som redan fanns i katalogen, utan istället ge den ett nytt namn. Därefter fick jag en prompt om att uppge lösenordet. När det var gjort hade jag filen **flag_1.txt** som innehöll flaggan.

<details>
  <summary><b>Klicka här för att se flaggan</b></summary>

  `THM{Cr4ck1n6_z1p$_1s_34$yyyy}`
</details>

## Jakt på nyckeln till Sidequest 2
Nu så, äntligen dags att jaga rätt på nyckeln till Sidequest 2!

- Sökte lite på måfå eftersom vi inte hade så mycket att gå på
- När jag kollade `sudo -l` på vår användare **ubuntu** stod det `ALL:ALL:ALL`
- Körde `sudo su` för att byta till root
- Jag kollade i `/etc/shadow` och tänkte att vi kunde knäcka lösenordet till root för att eventuellt få en ledtråd
- onind00 hittade däremot en spännande fil i hemkatalogen för **ubuntu**, nämligen `.Passwords.kdbx`

![Screenshot](img/Pasted%20image%2020251210170801.png)

- Jag kände inte till vad det var för fil så jag fick googla. Fick till svar att det är en databas-fil för lösenordshanteraren KeePass.
- Eftersom rummet handlade om att knäcka lösenord så tog jag för givet att vi skulle knäcka just den filen.
- Efter en sökning online fick jag reda på att **John the Ripper** kan knäcka såna filer, så länge man först skapar en hash som john kan förstå. Detta gör man med **keepass2john**.
- Alla installationsfiler till john fanns på **ubuntus** desktop, så jag tog mig till `/home/ubuntu/Desktop/john/run` där **keepass2john** fanns.
- Därefter körde jag `./keepass2john /home/ubuntu/.Passwords.kdbx > /home/ubuntu/keepass.hash` som i korta drag kör programmet och sparar dess output till filen **keepass.hash** i ubuntus hemkatalog
- Därefter var det dags för john att göra sin magi. Jag navigerade till hemkatalogen och körde kommandot `john --wordlist=/usr/share/wordlists/rockyou.txt keepass.hash`
- Det visade sig dock att jag fick ett felmeddelande om nån slags race condition.  

![Screenshot](img/Pasted%20image%2020251210172442.png)

- Jag tänkte att det kanske berodde på att jag var **root**. Jag bytte till användaren **ubuntu** igen och körde samma kommando, så funkade det.

<details>
  <summary><b>Klicka här för att se lösenordet</b></summary>

  ![Screenshot](img/Pasted%20image%2020251210172523.png)
</details><br>

- John visade sig från sin bästa sida igen! Jag blev dock lite fundersam till en början, för vad skulle jag med *ett enda* lösenord? Men det tog mig inte lång stund innan jag förstod att jag skulle använda det för att öppna själva databasen.
- En snabb sökning och jag fick veta att man kan öppna KeePass-filer med **keepassxc**, som av en händelse fanns på maskinen.
- Dock lirade det inte att öppna i terminalen - jag behöver en display.

![Screenshot](img/Pasted%20image%2020251210172823.png)

- Jag laddade ner `.Passwords.kdbx` till min Kali VM, installerade **keepassxc** på den och körde kommandot för att öppna filen. Det öppnade följande fönster:

![Screenshot](img/Pasted%20image%2020251210173113.png)

- I upplåst läge kunde jag se en nyckel men ingen intressant information. Jag klickade på fliken **Advanced** och såg att det fanns en png-fil.

![Screenshot](img/Pasted%20image%2020251210173305.png)

- Jag markerade filen och klickade på preview, då hittade jag den eftertraktade nyckeln till Sidequest 2

![Screenshot](img/Pasted%20image%2020251210173450.png)

- Nyckeln står i bilden, men jag håller den borta härifrån för spänningens skull ;)

## Prolog
Jag uppskattade rummet för att jag fick friska upp minnet när det kommer till att använda John the Ripper. Jag gillar verktyget, men jag har alltid tyckt att det har varit lite klurigt att använda eftersom man måste anpassa filerna så mycket för att john ska kunna läsa dem. 

Men det var framför allt den efterföljande jakten på nyckeln till Sidequest 2 som var riktigt spännande. Det är en så himla god känsla när man letar och letar, till slut hittar ett spår som leder en till mål. Jag lärde mig mer om hur databaserna till KeePass ser ut, hur man kan knäcka dem med john (förutsatt att det är ett svag lösenord som används...) och sedan öppna filen för att läsa dess innehåll. Grymt kul!