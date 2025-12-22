# RolePlay (Web)

![Screenshot](img/Pasted%20image%2020251215232818.png)

## Beskrivning

Som man kan läsa av utmaningens rubrik och beskrivning så verkar den handla om roller på något sätt. Alltså, baserat på vilken roll man blir tilldelad så har man access till olika saker. Jag kunde ju dock inte vara helt säker eftersom beskrivningen inte skriver ut det i klartext, så det var bara att hoppa vidare till den tilldelade URL:en och börja undersöka.

## Initial Recon

I beskrivningen fick man en länk som ledde till en hemsida där man kunde logga in eller skapa ett konto.

![Screenshot](img/Pasted%20image%2020251215153250.png)

Jag skapade ett konto och loggade in. Jag möttes av följande meddelande:

![Screenshot](img/Pasted%20image%2020251215153422.png)

Jag förstod alltså att jag antagligen hade blivit tilldelad en roll och att jag på något sätt behövde göra mig till **admin** för att hitta flaggan. Jag gjorde lite grundläggande research:

- Kollade source om det fanns nåt intressant i koden. 
	- Jag kunde konstatera att source endast innehöll HTML, lite styling, och i script-taggen endast JS som hanterade animationen i bakgrunden (text som hoppade omkring).
- Kollade om det fanns några ytterligare filer som laddades in.
	- Bootstrap laddas in - inte relevant.
	- Något som kallades `<anonymous code>` som jag inte riktigt vet vad det är

	  ![Screenshot](img/Pasted%20image%2020251215153832.png)
	  
- Inga ytterligare JS-filer laddades in.
- Kollade cookies
	- Här fanns en cookie med namnet `JWT_COOKIE`

	  ![Screenshot](img/Pasted%20image%2020251215153957.png)

För att se vad min JWT hade för egenskaper decode:ade jag den på **jwt.io**

![Screenshot](img/Pasted%20image%2020251215154130.png)

Det jag kunde konstatera var att den använder sig av algorithmen **ES256** (Elliptic Curve), och att min användare hade rollen **user**. Jag antog att det är den rollen jag vill få till **admin**.

## JWT Tampering

Genom att fånga mitt GET request i Burp kunde jag se min JWT, justera dess värde på **jwt.io** och sedan skicka ett nytt request med min nya payload med Burp Repeater. 

#### Alg none

Jag testade det mest grundläggande hacket - att ändra **alg** till **none**. Dock fick jag samma svar som när jag först loggade in.

![Screenshot](img/Pasted%20image%2020251215154615.png)

Jag testade även att helt sonika ändra role från user till admin, men det genererade samma respons.

#### Algorithm Confusion Attack

Jag läste mig till att man kan prova att utföra en [Algorithm Confusion Attack](https://portswigger.net/web-security/jwt/algorithm-confusion). Lite samma som när jag satte `alg` till `none`, men istället för att byta till `none` så sätter man en annan algorithm som ger en signatur på slutet. Till artikeln jag länkar till ovan nämns hur man kan byta från **RS256**, som använder en privat och en publik nyckel, till **HS256** som är en symmetrisk algorithm och därför endast använder *en* nyckel. Alltså samma för att kryptera och dekryptera.

Tanken är att man byter från **RS256** till **HS256**, och använder den *publika* nyckeln för **RS256** som "secret key" när man väl ändrat till **HS256**. Hänger ni med? Jag hänger typ med själv :D

Steg 1 för att detta ska funka är att hitta den publika nyckeln. Den kan finnas på ett antal olika kända endpoints, men en feroxbuster mot siten visade inga såna endpoints tyvärr. Nästa ställe att titta i är SSL-certifikatet.

Där hittade jag vad jag antog var den publika nyckeln för **RS256**, jag var dock aldrig helt säker på att så var fallet. Aja, man får testa, here goes nothing!

![Screenshot](img/Pasted%20image%2020251215191137.png)

Längst ner på bilden står ju de keywords jag vill ha - både *public key*, *Elliptic Curve* och *256*.

Verktyget jag ville använda var [jwt_tool](https://github.com/ticarpi/jwt_tool), och för att det skulle lira behövde jag den publika nyckeln i pem-format. Jag kunde alltså inte bara copy/paste från informationen ovan.

Genom att använda **openssl** kunde jag ladda ner den publika nyckeln och spara till fil: `openssl s_client -connect roleplay.appsec.nu:443 -showcerts 2>/dev/null | openssl x509 -pubkey -noout > public_key.pem`. Därefter kunde jag kontrollera nyckeln genom `openssl pkey -in public_key.pem -pubin -text` och då jämföra med vad jag såg i informationen om SSL-certifikatet.

![Screenshot](img/Pasted%20image%2020251215192823.png)

Därefter använde jag **jwt_tool** för att skapa mig en ny JWT med alg satt till **HS256**, role till **admin** och den publika nyckeln som **secret**. När det var klart skickade jag med min nya JWT i requesten via Burp. Om det funkade? Inte alls :D

Jag testade massor av roller, även om jag innerst inne var övertygad om att det skulle vara just role admin...
## Genombrottet
Man skulle kunna tro att det här var första försöket, men icke. Jag hade gjort precis samma steg som ovan någon kväll tidigare, och den här gången tillsammans med mina lagkamrater. Ett misslyckande kan ju vara handhavarfel. Två misslyckanden? Då är det nog inte rätt väg...

Back to basics. **Onind00** läste instruktionen igen. Noga. Han fick oss andra att läsa den igen, ännu mer noga. Vi lade märke till att det stod nåt om *psychic abilities*. Kanske man skulle byta roll till nåt mer andligt?

**Onind00** kollade source på siten igen. Lade märke till en avslöjande kommentar längst ner på sidan:

![Screenshot](img/Pasted%20image%2020251215193436.png)

En googling senare på OpenJDK 17 och JWT gav oss the holy grail. Det visade sig att det finns en CVE som kallas för **Psychic Signatures**. Oj oj. Här hade vi testat manuella grejer, och så fanns det en CVE hela tiden?? [CVE-2022-21449](https://nvd.nist.gov/vuln/detail/cve-2022-21449)

## Psychic Signatures

Vi började genast läsa på olika håll. På Nist stod inte namnet Psychic Signatures uttryckligen, men jag hittade [ett blogginlägg](https://www.securecodewarrior.com/article/psychic-signatures) som tog upp just detta. Jag läste inlägget och fastnade för exploit-delen:

![Screenshot](img/Pasted%20image%2020251215194113.png)

Jag hoppade över till Burp och redigerade min JWT med **role admin**, suddade ut signaturen och lade till `MAYCAQACAQA` istället och skickade iväg. Funkade det? Ja :D

![Screenshot](img/Pasted%20image%2020251215194301.png)

Stack över till CTF-portalen och submittade flaggan. Error. Va? Det kan inte vara sant. Tryckte på **Render** i Burp istället och såg att den råa responsen innehöll lite skräptecken - här är den riktiga flaggan:

![Screenshot](img/Pasted%20image%2020251215194447.png)

## Slutord

En utmaning som på pappret inte skulle vara så svår, men oj vad tid det tog för oss. Det gäller ju att man ska veta vad man ska göra och hur. Nästa gång jag/vi står inför JWT-utmaningar så har vi ett par metoder att vända oss till, bra det.

Denna flagga är ännu ett bevis på att teamwork är dreamwork. Jag hade kört ner mig i mina hjulspår och det krävdes en lagkamrat för att knuffa mig lite i sidled och tänka om.

Den här flaggan var riktigt härlig att fånga hem till slut :)