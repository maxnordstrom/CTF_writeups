# Silent Monitor

*2026-06-08*

![Screenshot](img/Pasted%20image%2020260608123512.png)
## Intro

#### Green Lights, Dark Corners

> CorpNet's internal network operations centre has been running quietly for years. Monitoring hosts, logging events, and keeping the infrastructure alive. Or so it seems. A tip from a disgruntled contractor suggests that someone on the NOC team has been cutting corners, leaving doors open, and hiding things in places no one thinks to look.
> 
> The portal is up. The services show green. The audit log looks clean.
> 
> But clean logs can be written by anyone.
> 
> Your job is to get in, move through the system, and find out what is really running behind the secret dashboard.

![Screenshot](img/Pasted%20image%2020260608123718.png)
## Recon

Nmap:ade IP:t och den svarade med att port 22 och 5050 är öppna. Lade till flaggan `-sV` för att få ut lite mer info om just 5050 och fick följande svar:

![Screenshot](img/Pasted%20image%2020260608124409.png)

HTTP, let's go!
### Webben

![Screenshot](img/Pasted%20image%2020260608124711.png)

Inget att klicka på, inget i källkoden, feroxbuster gav inga träffar.

Lade till corpnet.thm i `/etc/hosts` för att se om det hjälpte, men inte direkt. Det var lite av en chansning.

Fuzza för subdomäner kanske? Kankse sen.

Efter lite mer trixande med **fuff** hittades endpointen `/internal`, nice.

![Screenshot](img/Pasted%20image%2020260609093038.png)

Testade `'OR 1=1--;` i username och det verkar som att jag redirectas.

![Screenshot](img/Pasted%20image%2020260609093419.png)

Fångade requesten i Burp Proxy istället för Repeater och modifierade requesten on the go. Efter det kom jag till dashboarden inloggad som användaren `netops`.

![Screenshot](img/Pasted%20image%2020260609093745.png)

I `/internal/health` ska man kunna pinga olika adresser för att kolla status.

![Screenshot](img/Pasted%20image%2020260609094357.png)

När jag skriver in serverns IP får jag respons som ser trovärdig ut, och när jag skriver in en random privat IP får jag 100% packet loss, så det ser ju faktiskt ut som att det kör på riktigt, inte bara nåt för syns skull.

Det visar sig att vi kan skicka in fler kommandon till servern efter ping-kommandot. Detta avslöjas i översikten på sajten:

![Screenshot](img/Pasted%20image%2020260609100430.png)

När jag skickade följande i Burp Repeater fick jag en trevlig träff :)

![Screenshot](img/Pasted%20image%2020260609100527.png)

Vad finns i `secret.config`?

![Screenshot](img/Pasted%20image%2020260609100759.png)

Credentials! Till SSH? Ja :) Så första flaggan är i hamn.
## Flagga 2

I hemkatalogen för `sysadmin` finns en katalog som heter `backups`. I den finns `infrastructure.kdbx` som är en databasfil för Keepass, och om jag inte missminner mig finns det ett program som kan öppna dem mer eller mindre smärtfritt.

Kollade mina gamla anteckningar och hittade en notering från jul-utmaningarna 2025. John the Ripper kan ordna biffen.

Men, när jag körde `keepass2john` fick jag följande felmeddelande:

![Screenshot](img/Pasted%20image%2020260609101749.png)

Verkar som att John inte har stöd för det formatet. Google säger att jag kan testa keepass2john.com istället, intressant. Men där får jag också error.

Lite sökningar på nätet ger mig inga nya leads på det spåret. Dags att kolla eskalering på annat vis - kan ju hända att vi inte behöver keepass-filen.
### Eskalering

Får inte köra sudo

![Screenshot](img/Pasted%20image%2020260609102800.png)

`find / -type f -perm -04000 -ls 2>/dev/null` gav inga intressanta träffar på binärer med s-biten satt.

I `/opt` finns katalogen `/netops`. Den har vi inte permissions till som användaren `sysadmin`, men `www-data` har access, så tar en titt via buggen på hemsidan istället. Visade sig att det är just där t.ex. `secret.config` ligger. Dags att titta närmare på resten av filerna.

![Screenshot](img/Pasted%20image%2020260609103524.png)

Eftersom vår användare inte har access till katalogen fick jag kopiera databasen till `/tmp`, startar därefter en python-server och drar ner den till min lokala burk.

Men det blev ju 404 eftersom `sysadmin` inte har rättigheter för filen. Starta python-server som `www-data`?

![Screenshot](img/Pasted%20image%2020260609104942.png)

Lite svårigheter med att förstå var python-servern startas dock.

![Screenshot](img/Pasted%20image%2020260609125525.png)

Mina vänner hittade root-flaggan, men den hade inte jag tid att fokusera på, jag ville verkligen lyckas att ladda ner databasen :D

Provade väldigt många olika payloads från revshells.com, men stötte konstant på två olika problem. Antingen funkade de inte på servern, eller så blockerades mina payloads av nån WAF eller liknande.

Tobzon hjälpte mig att konstatera att `&`, oavsett i klartext eller URL-encoded, blockerades. Därför kom idén att skicka en payload i base64, decoda den och därefter exekvera. När jag gjorde detta i ett svep - i ett försök att starta ett rev shell utan att skriva något till servern - misslyckades jag ändå. Kan WAF:en ha snappat upp det ogiltiga tecknet ändå?

Jag hade tagit reda på att jag kunde skapa nya filer och skriva till dem som `www-data` genom buggen på websidan. Så en ny idé tog form:

- Skapar en fil, typ `/tmp/hej.txt`
- Skriver en payload till den i base64 genom webben - inga ogiltiga tecken
- Kör `base64 -d /tmp/hej.txt > /tmp/rev.sh`
- Gör filen körbar med chmod
- Skickar payloaden `bash /tmp/rev.sh` och snappar upp shellet i min lokala lyssnare.

Funkade det? Ja! En otrolig seger. Nu kunde jag äntligen, utan svårighet, ladda ner databasen `netops.db`. Innehöll den nåt viktigt? Absolut inte, men det spelade ingen roll :D

![Screenshot](img/Screenshot%20from%202026-06-09%2012-31-33.png)
## Root-flaggan

Det visade sig att alla våra omvägar ledde tillbaka till keepass-filen och John the Ripper. Vad som skulle till var att ladda ner en annan version av John, en som kan hantera den nya varianten av keepass-filer.

Fick ladda ner och installera versionen `bleeding-jumbo` av John the Ripper.

Senaste versionen av `keepass2john` kom med, så det var bara att köra. Hanterade .kdbx-filen galant vilket gav mig en hash som john förstår.

Körde john med rockyou och fick en tidig träff:

![Screenshot](img/Pasted%20image%2020260609151326.png)

Använde lösenordet för att öppna keepass-filen med `KeePassXC` och där hittade jag lösenordet till root:

![Screenshot](img/Pasted%20image%2020260609151458.png)

SSH in på servern som `sysadmin` (för jag kunde inte SSH:a som root?), `su root` och så var det bara att hämta hem flaggan. Gött!

## TL;DR

Base64-encoda era framtida payloads ;) Happy hacking!