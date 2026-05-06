![Screenshot](img/Pasted%20image%2020260504162615.png)

https://tryhackme.com/room/include

![Screenshot](img/Pasted%20image%2020260504162636.png)

> _"Even if it's not accessible from the browser, can you still find a way to capture the flags and sneak into the secret admin panel?"_

Alright, server-side attacks here we go! Två flaggor att hämta, låt oss börja med en gammal hederlig nmap och kolla vad som går att hitta.
## Recon

En snabb portscan mot alla tcp-portar gav träff på 7 öppna. En lite mer noggrann scanning av dessa 7 gav följande resultat:

![Screenshot](img/Pasted%20image%2020260504221302.png)

Tur att jag inte stängde av den första scanningen allt för tidigt, för innan den hunnit i mål hittade den även port 50000 öppen. En närmare titt på den gav:

![Screenshot](img/Pasted%20image%2020260504221608.png)

Oh la la. En hemsida kanske? Börjar med att ta mig en närmare titt på den.
## Port 50000

![Screenshot](img/Pasted%20image%2020260504221844.png)

Utöver den förmanande texten har jag möjlighet att logga in.

![Screenshot](img/Pasted%20image%2020260504222004.png)

`admin:admin` funkade inte :D

Hittar en liten kommentar i slutet av login-sidan som kan vara av intresse.

![Screenshot](img/Pasted%20image%2020260504222137.png)

Login-sidan heter `/login.php`, så varför inte köra en ffuf mot sajten och se vad som dyker upp?

Hittade några trevliga directories:

![Screenshot](img/Pasted%20image%2020260504223619.png)

Vad trevligt att modulen har handlat om template injection, och så finns det en passande mapp som heter just templates. Som av en händelse :D (Visade sig att modulen inte alls hade handlat om template injection, det var visst en annan...)

![Screenshot](img/Pasted%20image%2020260504223720.png)

Mest bootstrap och jquery, men några custom-grejer.

`footer.php` innehåller just den kommentaren jag lade märke till på login-sidan:

![Screenshot](img/Pasted%20image%2020260504223829.png)

Känns ju som att jag på nåt vis vill injicera nåt där på slutet. Kanske?

`header.php` innehåller, ja, headern. Med meny o.s.v.

![Screenshot](img/Pasted%20image%2020260504224001.png)

`/javascript` ger kod 403. `/uploads` däremot:

![Screenshot](img/Pasted%20image%2020260504224145.png)

En urtjusig profilbild.

<img height="200px" src="Pasted%20image%2020260504224209.png">

Kör en gobuster och letar specifikt efter php-filer.

![Screenshot](img/Pasted%20image%2020260504225842.png)

## Port 25

Åtminstone 5 andra portar har ju med mail att göra - vore konstigt om jag inte kunde hitta nåt smaskigt där. Här kommer en refresher över vad jag hittade tidigare:

![Screenshot](img/Pasted%20image%2020260504221302.png)

[Pentestmonkey](https://pentestmonkey.net/tools/user-enumeration/smtp-user-enum) har ett trevligt litet verktyg för att enumerera användare på smtp. Detta finns redan installerat på Kali, så vad väntar vi på?

`smtp-user-enum -M VRFY -U /usr/share/seclists/Usernames/top-usernames-shortlist.txt -t $IP` 

![Screenshot](img/Pasted%20image%2020260505162529.png)

Gav mig inga användarnamn jag inte hade kunnat gissa mig till. Kör med "vanliga" namn också.

Då hittar jag `bin`, ännu ett system-namn, men även `charles`. Tar sjukt lång tid för jag satte 5 sekunder timeout på varje query, och listan jag använder nu har 10.000 entries... Startar **enum4linux** parallellt.

Enum4linux avslutades för att servern inte tillåter att username och password är tomma. Testar att skicka med `charles` men det lirade inte då heller.

**smtp-user-enum** är klar

![Screenshot](img/Pasted%20image%2020260505170738.png)

Så `charles` och `joshua` är ju verkligen av intresse. 

Kan vi ansluta?

![Screenshot](img/Pasted%20image%2020260505163116.png)

![Screenshot](img/Pasted%20image%2020260505163204.png)

## POP3 och IMAP

Jag är ju taggad på mail. Så en hydra med mina giltiga användarnamn mot POP3 och IMAP?

Sätter upp Hydra mot port 995.

Aj, aj, aj, Charles...

![Screenshot](img/Pasted%20image%2020260505172630.png)

Och `joshua` hade även han 123456. Det luktar ju false positive lång väg. Men värt att kolla.

Nämen!

![Screenshot](img/Pasted%20image%2020260505173013.png)

Men

![Screenshot](img/Pasted%20image%2020260505173304.png)

![Screenshot](img/Pasted%20image%2020260505173253.png)

Helt tomt i inkorgarna...
## Port 4000

Enligt namp körs Node.js med Express. Vad säger curl? Oj oj, nåt slags inlogg. Kollar i browsern.

![Screenshot](img/Pasted%20image%2020260505171324.png)

Man är väl inte sämre än att man gör som man blir tillsagd? :D `guest:guest` it is!

![Screenshot](img/Pasted%20image%2020260505171456.png)

![Screenshot](img/Pasted%20image%2020260505173513.png)

Två fält på min profil där jag kan rekommendera aktiviteter. Mjau. Det jag skriver läggs in i listan ovanför.

![Screenshot](img/Pasted%20image%2020260505173659.png)

Körde en ferox och skickade med min `connect.sid` header och kollade lite, inget mega-häftigt.

![Screenshot](img/Pasted%20image%2020260505174802.png)

Misstänker att det kan finnas nån form av sårbarhet för Property Pollution i det här rekommendera-aktivitet-formuläret.

Testar lite olika payloads utan att lyckas.

Testade dock den lite enklare varianten att bara skriva in "isAdmin" och "true" i fältet och skickade in. Då ändrades `isAdmin` till `true`. Två nya sidor visar sig:

![Screenshot](img/Pasted%20image%2020260506095817.png)

Menyvalet API visar:

![Screenshot](img/Pasted%20image%2020260506095907.png)

Och Settings visar:

![Screenshot](img/Pasted%20image%2020260506100451.png)

Här kan man ladda upp en banner som renderar på profil-sidan. Vore ju trevligt att ladda upp en maskerad php-fil som läser filer på servern istället :)

Eller nej, vi vill ju sätta adressen så att den kallar på det interna API:et såklart!

Pekade URL:en till getAllAdmins-API:et och fick följande:

![Screenshot](img/Pasted%20image%2020260506101017.png)

Ett svar med en lång base64-sträng. Trevligt :)

En cyberchef senare:

![Screenshot](img/Pasted%20image%2020260506101153.png)

```json
{
	"ReviewAppUsername":"admin",
	"ReviewAppPassword":"admin@!!!",
	"SysMonAppUsername":"administrator",
	"SysMonAppPassword":"S$9$qk6d#**LQU"
}
```

Första flaggan ska finnas på SysMon-sajten, så går till sajten på port 50000 och loggar in med `administrator:S$9$qk6d#**LQU` och flaggan är i hamn!

<details>
  <summary><b>Klicka här för att se den</b></summary>

  ![Screenshot](img/Pasted%20image%2020260506101812.png)
</details>

## Flagga två

Den ska hittas genom att läsa en hemlig fil på `/var/www/html`

Vad kan vi hitta för trevligt på SysMon-sajten till att börja med?

Till vår stora glädje hittar vi en URL med en parameter i koden för `/dashboard.php` Den pekar till `/profile.php?img=profile.png` vilket känns som en utmärkt ingång för LFI.

Gjorde några manuella tester i Burp för att kolla LFI. Först - kolla om koden rensar bort `../` i ett försök att stoppa path traversal.

I Burp testade jag helt enkelt att peka mot `../profile.png` istället för endast `profile.png`. Bilden visades ändå, vilket tyder på att `../` plockas bort.

Tricket blir att skriva `....//` istället, då kommer den otillåtna kombinationen av tecken att rensas bort, men faktum är att det som blir kvar då är just `../` :D

Försökte jaga rätt på `/etc/passwd`, och det krävdes hela nio stycken `....//` för att den skulle peka rätt, men det blev rätt till slut!

![Screenshot](img/Pasted%20image%2020260506115945.png)

Försökte istället peka till den hemliga filen i `/var/www/html` och gissade på att den skulle heta `flag.txt`, men icke. Hur skulle vi kunna hitta filnamnet?

## SSH

Det här var ju en blunder av rang. Jag hade tidigare hittat giltiga credentials till pop3, varför testade jag inte dem på ssh?

Det visade sig att de kunde logga in lätt som en plätt, och skriva ut `/etc/passwd` på ett lite enklare sätt :D

![Screenshot](img/Pasted%20image%2020260506120246.png)

Och tror ni inte att joshua kunde hitta den hemliga filen?

![Screenshot](img/Pasted%20image%2020260506120340.png)

<details>
  <summary><b>Flaggan flaggan!</b></summary>

  ![Screenshot](img/Pasted%20image%2020260506120512.png)
</details>



Faktum är att väl inne i `/var/www/html` kunde vi skriva ut `dashboard.php` och hitta första flaggan som var skyddad bakom inlogget :D

<details>
  <summary><b>Klicka här för att se buset</b></summary>

  ![Screenshot](img/Pasted%20image%2020260506144956.png)
</details>

Så, vi hade egentligen access till båda flaggorna redan efter att ha hittat de båda användarnamnen och giltiga lösenord. Där ser man :)

## Slutord

Den första flaggan hittades genom att först utnyttja en Mass Assignment-sårbarhet i "recommend activity"-formuläret, där jag kunde sätta `isAdmin: true` på min egen användare. Jag trodde att sårbarheten skulle handla om Prototype Pollution, men det här var snarlikt. Därefter fick vi tillgång till fler sidor, där vi hittade ett lokalt API och kunde utnyttja en SSRF-sårbarhet via ett formulär och anropa API:et och få inloggningsuppgifter till svar. Det känns som att det var helt by the book.

Den andra flaggan hittade vi, men det känns som att det var på ett sätt som inte var helt i linje med de teman modulen hade tagit upp. Därför kollade vi in några writeups för att se hur andra löst uppgiften.

Som sagt, vi hittade LFI-sårbarheten, men visste inte riktigt hur vi skulle utnyttja den för att hitta den hemliga filen.

[Denna write-up](https://medium.com/@z0diac/include-ctf-tryhackme-writeup-c36bded6d2f4) beskriver riktigt snyggt hur man kan använda sig av Log Poisoning med hjälp av mail-loggarna. Kortfattat handlar det om att med hjälp av LFI hitta filen `/var/log/mail.log`. Denna fil har i regel både läs- och skrivrättigheter.

Genom att skicka ett mail via telnet eller nc till en av de identifierade användarna fylls `mail.log` med information om den aktiviteten. Istället för att uppge en giltig avsändaradress kan man istället skriva `<?php system($_GET['cmd']); ?>` som då kommer att exekveras när man pekar till `mail.log` via LFI.

Här är [en annan writeup](https://medium.com/@S1L3NT37/include-tryhackme-detailed-writeup-11c74c7ac7e1) där man får följa en person som i princip hade samma tillvägagångssätt som vi, men som även lade till log poisoning på slutet. 

Riktigt intressant läsning båda två. Happy hacking!