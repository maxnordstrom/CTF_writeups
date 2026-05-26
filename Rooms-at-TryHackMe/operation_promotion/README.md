![Screenshot](img/Pasted%20image%2020260522095925.png)

https://tryhackme.com/room/operationpromotion
## Introduction

> You are up for promotion at **Hadron Security**. Your senior lead, Mara, has handed you a solo engagement against **RecruitCorp**, a small recruiting firm with a public-facing portal. Compromise the host, capture the flags, and demonstrate that you are ready for the Penetration Tester title.

![Screenshot](img/Pasted%20image%2020260522100019.png)
## Recon

Fyra öppna portar:

![Screenshot](img/Pasted%20image%2020260522100643.png)

Företaget heter **RecruitCorp** och hemsidan ser ut så här:

![Screenshot](img/Pasted%20image%2020260522100751.png)

Ser en mailadress som kan komma till användning senare: `careers@recruitcorp.thm`

Feroxbuster gav ett gäng träffar:

![Screenshot](img/Pasted%20image%2020260522101123.png)

Robots.txt verifierar endast att endpointen `/admin` finns

![Screenshot](img/Pasted%20image%2020260522101042.png)

På adminsidan möts jag av ett inlogg:

![Screenshot](img/Pasted%20image%2020260522101145.png)

Inget hemligt i källkoden. 

Min fuzz visade även `/admin/users` som innehåller:

![Screenshot](img/Pasted%20image%2020260522103416.png)

När jag klickar på `lookup.php` redirectas jag till `/admin`.

Då känns det som att jag ska titta vidare på Samba-grejerna i jakt på credentials. Testar **smbclient**

`smbclient -L //$IP -N`

![Screenshot](img/Pasted%20image%2020260522101616.png)

`smbclient //$IP/public -N`

![Screenshot](img/Pasted%20image%2020260522101735.png)

Hittade `README.txt` men den innehöll inget skoj

![Screenshot](img/Pasted%20image%2020260522102203.png)

Får se vad **enum4linux** kan ge för nåt.

Hittar ett användarnamn som sticker ut: `jford`

Hittar även en custom domän på port 139:

![Screenshot](img/Pasted%20image%2020260522103337.png)

Och att lösenorden har en dåligt policy och är giltiga väldigt länge...

![Screenshot](img/Pasted%20image%2020260522103733.png)

Med tanke på detta kan det vara värt att köra en Hydra mot admin-inlogget

`hydra -l jford -P /usr/share/wordlists/rockyou.txt 10.114.171.112 http-post-form "/admin/:username=^USER^&password=^PASS^:F=Invalid"`

Det gav ingen träff.

SQL injection på admin-inlogget?

![Screenshot](img/Pasted%20image%2020260522105910.png)

Javisst, det lirade!

![Screenshot](img/Pasted%20image%2020260522105931.png)

Och nu kan vi komma åt `lookup.php` add döma av formuläret.

När jag klickar på "View own profile" får jag veta att `jford` har ID 1. Vem har 2? (Sen visade sig att jag inte alls är `jford`, utan user ID 1 är `admin`)

![Screenshot](img/Pasted%20image%2020260522110141.png)

Men det blir ju segt att lista alla en och en. Dumpa alla på en gång?

Kunde inte hålla mig så jag körde ett gäng manuellt. Spännande på `ID 7`

![Screenshot](img/Pasted%20image%2020260522110646.png)

Om jag följer URL:en får jag följande:

![Screenshot](img/Pasted%20image%2020260522110735.png)

Lade till IP:t på maskinen bara för att se output, fick följande:

![Screenshot](img/Pasted%20image%2020260522110821.png)

Vad händer om man lägger till `;id` efter IP:t?

![Screenshot](img/Pasted%20image%2020260522110918.png)

Hittar vår berömda `jford`

![Screenshot](img/Pasted%20image%2020260522111325.png)

Men jag kan inte läsa i hans hemkatalog eftersom jag är `www-data`

Letade vidare och hittade en intressant config-fil

![Screenshot](img/Pasted%20image%2020260522111524.png)

Börjar bli trött på att navigera i browsern, vill sätta upp en reverse shell istället. [Revshells.com](https://revshells.com) är min go-to och testar några olika one-liners därifrån.

Det tog en lång tid innan jag hittade ett som funkade, men det landade till slut på python3

![Screenshot](img/Pasted%20image%2020260522132826.png)

Lite trevligare att navigera i terminalen

![Screenshot](img/Pasted%20image%2020260522132910.png)

Uppgraderade mitt shell så det blev lite mer stabilt. I mitt shell:

`python3 -c 'import pty; pty.spawn("/bin/bash")'`

Därefter trycka Ctrl+Z, sen:

`stty raw -echo; fg` 

Enter två gånger, sen `export TERM=xterm` Done.

Nu är ju frågan hur jag ska eskalera...

Letade runt på systemet och valde till slut att söka efter databasen - den måste ju ligga på systemet.

![Screenshot](img/Pasted%20image%2020260522152101.png)

`app.db` är ju högst intressant.

Att läsa ut den med **cat** funkade inte, så jag öppnade den med **sqlite3** istället. Där finns tabellen `users` som jag misstänkte, och voilà, det går att läsa lösenorden i klartext.

![Screenshot](img/Pasted%20image%2020260522152231.png)

Dock ingen användare för `jford`... Och adminlösenordet funkar endast för att logga in i admin-portalen på hemsidan, det lirar inte att köra `su jford` och uppge det lösenordet...

Tittar lite närmare på samba-grejen

![Screenshot](img/Pasted%20image%2020260522155230.png)

Men det var en dead end.

Nu blev jag trött på att leta manuellt, så laddade upp **linpeas** på servern som fick göra sitt jobb:

![Screenshot](img/Pasted%20image%2020260522161824.png)

Hittade en trevlig grej:

![Screenshot](img/Pasted%20image%2020260522162040.png)

Cronjob att utnyttja?

![Screenshot](img/Pasted%20image%2020260522162523.png)

Sockets?

![Screenshot](img/Pasted%20image%2020260522162738.png)

Lite för mycket info att ta in, och inget som direkt ropar "Hej, här är vägen in!". Googlade på `CVE-2025-38236`, men handen på hjärtat, jag hängde inte riktigt med där...

## Lösenordet
**Onind00** tog sig en titt på lösenordshashen vi hittat config-filen för databasen, alltså `$2b$10$QzkXmGndA2cQLozO3xAN6eWKrl6ZXyzhYTJNF67exOmTmN5oVSEfq`

Vi hade redan satt igång hashcat med rockyou.txt utan resultat, men genom att skapa en ny lösenordslista baserat på keywords som hittats på hemsidan (ca 6000 kandidater) blev det träff! Med rätt lösenord kunde vi byta användare till den berömda `jford`

![Screenshot](img/Pasted%20image%2020260526121156.png)

I hemkatalogen hittas `user.txt` som innehöll flaggan!
## Nästa flagga
Var kan vi hitta `flag.txt`? Antagligen i root-katalogen.

Av en händelse får vi köra **find** med root-behörigheter

![Screenshot](img/Pasted%20image%2020260526121526.png)

Och då hittades flaggan:

![Screenshot](img/Pasted%20image%2020260526121730.png)

Enligt GTFOBins kan find ge oss ett root-shell:

![Screenshot](img/Pasted%20image%2020260526121840.png)

Voilà!

![Screenshot](img/Pasted%20image%2020260526122048.png)

Och då var det en baggis att läsa ut flaggan.

Tänk att det var ett litet lösenord som var det som hindrade oss under så lång tid... Lärdom som jag tar med mig: testa alltid med en custom wordlist!

Happy hacking! :)