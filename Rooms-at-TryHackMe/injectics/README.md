# Injectics

![Screenshot](img/Pasted%20image%2020260424171310.png)

https://tryhackme.com/room/injectics

Det sista rummet i modulen *Injection Attacks* i pathen *Web Application Pentesting*. Det består endast av två frågor:

![Screenshot](img/Pasted%20image%2020260424171413.png)

Jag har startat maskinen och fått en IP-adress, so let's go!
## Initial recon
Inleder med en snabb och bred nmap scan genom `nmap -p- $IP -v`

Rätt snabbt ser jag att port 80 är öppen, så innan scanningen är helt klart öppnar jag webbläsaren för att se om det finns en hemsida. Rummet handlar ju trots allt om injection och att logga in på en adminpanel :)

Hemsida finnes.

![Screenshot](img/Pasted%20image%2020260424172152.png)

Det finns undersidor för events, atleter, kontakt m.m. men alla är inaktiverade. Den enda undersidan som funkar är *Login*. Spicy.

![Screenshot](img/Pasted%20image%2020260424172336.png)

Första flaggan ska jag få genom att logga in som admin. Eventuellt att man kan hitta ledtrådar om man lyckas logga in som vanlig användare, men det verkar inte finnas nåt sätt för att registrera sig. 

Sidan för inloggning som admin är i princip identisk rent grafiskt, bara att knappen saknas.

![Screenshot](img/Pasted%20image%2020260424172916.png)

Det som visar att man försöker logga in som admin är att URL:en pekar mot `adminLogin007.php` istället för `login.php`

Dags att leta sårbarheter kopplade till injection.
## Scanning for injection vulns

Innan jag drar igång nån häftig scanner testar jag några basic tricks, typ `' OR 1=1--`

Startade Burp så jag enkelt kan skicka iväg några varianter med Repeater.

Dock vore det trevligt att ha en giltig e-postadress för detta, så jag kickar igenom om det finns nån rolig kommentar i källkoden eller annan fil. Det visade sig vara inte helt dumt - på startsidan kunde jag hitta detta:

![Screenshot](img/Pasted%20image%2020260424173647.png)

Jag kommer utgå från att `dev@injectics.thm` är en giltig mailadress och skriva min payload i lösenordsfältet

Tricket med `' OR 1=1--` svarar dock endast med invalid email or password.

![Screenshot](img/Pasted%20image%2020260424174041.png)

Jag testar några olika varianter, både i fältet för användarnamn fältet för lösenord. Verkar dock inte hitta nåt som funkar. Här är några jag testat:

![Screenshot](img/Pasted%20image%2020260424175213.png)

![Screenshot](img/Pasted%20image%2020260424175242.png)

![Screenshot](img/Pasted%20image%2020260424175326.png)

![Screenshot](img/Pasted%20image%2020260424175407.png)

![Screenshot](img/Pasted%20image%2020260424175514.png)

![Screenshot](img/Pasted%20image%2020260424175552.png)

Jag ska prova min lycka med någon av de ovanstående, fast för inloggningen för vanliga användare.

Lite intressant där är att webbläsaren inte skickar en POST till `login.php` utan till `functions.php`. Detta verkar dock endast ha att göra hur de satt upp sajten. Att `login.php` använder JS istället för en HTML form submission.

Efter två försök så fick jag en respons som jag tycker om:

![Screenshot](img/Pasted%20image%2020260424181358.png)

Responsen verkar tyda på att min injection har lyckats och att den försöker redirecta mig till `dashboard.php?isadmin=false`. Om jag försöker skriva in min payload på själva hemsidan får jag dock en varning om att jag använder otillåtna tecken...

Endpointen `dashboard.php` verkar heller inte finnas, så kör en Feroxbuster för att vara säker. Kanske hittar nåt annat gott på vägen.

Förutom massa endpoints för phpmyadmin hittade jag `/flags`, men den har jag inte behörighet att visa. Dock ingen `dashboard.php`...

Men nu som ett under när jag laddade om sidan, klickade på menyvalet för Login så redirectades jag till `/dashboard.php` i inloggat läge! (Okej, det var kanske inget under, bara jag som glömt av lite hur Burp lirar :D ) 

Det var min payload med `dev@injectics.thm'#` som funkade. Rätt email, och sen stängs statementet med `'` och resten kommenteras ut med `#`

![Screenshot](img/Pasted%20image%2020260424182921.png)

Jag hittar dock ingen flagga, så frågan är om jag är inloggad på rätt ställe. 

![Screenshot](img/Pasted%20image%2020260424183037.png)

Tetade att skriva in `dashboard.php?isadmin=true` (och false) men det gör ingen skillnad. Mystiskt.
## mail.log
Min gode vän Onind00 gjorde en spännande upptäckt som eventuellt kan leda till nåt spännande. Jag hade ju själv hittat en kommentar i källkoden om att mail sparas till `mail.log`. Detta trodde jag var en fil/endpoint man inte kunde nå från utsidan, men icke! Hans fuzzing (men inte min, buhu) av endpoints hade gett en träff på just den. Vad gömmer sig där?

Oj oj oj.

![Screenshot](img/Pasted%20image%2020260424201422.png)

Med andra ord. Om (läs *när*) databasen blir raderad så kommer det insertas två nya konton med default credentials enligt listan. Detta så att de aldrig förlorar åtkomst. Hehe. Safety first, eller?

Så en liten SQLi som råkar radera databasen vore på sin plats - då är det ju bara att logga in med uppgifterna ovan.

Obs. Jag testade att logga in med uppgifterna direkt utan SQLi - de verkar inte vara aktiva för tillfället.

Då tror jag på följande attackkedja: Ta sig förbi inloggningen för vanliga användare med SQLi så jag loggar in som  `dev`, därefter SQLi i formuläret där man kan redigara ledartavlan. Den injektionen raderar databasen och utvecklarens funktion som automatiskt lägger till standardkonton triggas. Då borde jag komma åt första flaggan.
## Ledartavlan

Testade att använda min payload på "superadmin-kontot", det funkade lika bra det. Nu välkomnas `admin`, men ingen ytterligare funktionalitet finns.

![Screenshot](img/Pasted%20image%2020260424212803.png)

Jag har roat mig med att deface:a alla stats, så allt står på 0. Men jag tänker att det kanske är här det finns en möjlighet att radera databasen helt.

När jag går in och redigerar en post i ledartavlan skickas följande POST request till `edit_leaderboard.php` 

`rank=1&country=&gold=1&silver=2&bronze=3`

Lite spännande är att det finns en variabel som heter `country` och att den är tom. Kanske en möjlig ingång för SQLi?

Kopierade i alla fall hela requesten jag fångat i Burp, sparade den till fil och använde med sqlmap enligt följande:

`sqlmap -r request.txt --level=3 --risk=2 --dbs`

Tyvärr inget lyckat resultat just med det kommandot. Men jag passade på att göra en körning mot fältet för användarnamn - där jag redan vet att det går att ta sig förbi utan lösenord med SQLi. Lämande dock datorn lite för länge, så maskinen hann få slut på tid, men det sista som sqlmap hade returnerat var det här:

![Screenshot](img/Pasted%20image%2020260425005707.png)

Så, troligtvis att MySQL används (var min gissning från första början visserligen), och eventuellt att man kan radera databasen från inloggningsformuläret? Provar mer imorgon.

![Screenshot](img/Pasted%20image%2020260426164749.png)
## Ny dag ny maskin

Körde igång sqlmap igen och det kan bekräfta att fältet för användarnamn är sårbart för SQLi (vilket jag förvisso redan visste) och gav lite mer bra info:
- Server OS: Ubuntu
- Stack: Apache med MySQL 5.0.12

![Screenshot](img/Pasted%20image%2020260425101525.png)

Så jag vet att man kan köra en `SLEEP(5)` och att servern väntar med responsen - då borde det ju gå att radera alla databaser också.

Visade sig att det inte går att wildcard:a på det sättet med MySQL. Jag behöver namn på en databas eller tabell.

Dock kan sqlmap skicka request till den nuvarande databasen utan att uppge ett namn, så även om jag inte lyckats enumerera namnen på tabellerna skulle jag kunna prova att radera tabeller som vanligtvis brukar finnas.

Testade därför med `sqlmap -r req.txt --flush-session --sql-query="DROP TABLE users;"`, fick dock det här svaret:

![Screenshot](img/Pasted%20image%2020260425111421.png)

Då är det nog som jag tänkte från början -  inloggningsformuläret är vägen in, men ledartavlan är nog stället som ska utnyttjas för att radera databasen.
## Ledartavlan igen

Fångade POST requesten för ledartavlan och körde sedan `sqlmap -r leaderboard.txt --flush-session --level=3 --risk=3 --dbs`

Hittar inget spännande, så ökar level till 5 och gör en ny körning.

Det gav en träff på parametern `gold`!

![Screenshot](img/Pasted%20image%2020260425113306.png)

Tillbaks till Burp för att se om jag kan droppa tabellen users. Users borde ju finnas.

![Screenshot](img/Pasted%20image%2020260425113446.png)

I Burp står det `Error updating data`, men när jag laddar om sidan ser jag att guldmedaljerna för övriga länder är borta. Kanske lyckats ändå?

![Screenshot](img/Pasted%20image%2020260425113623.png)

Lyckas inte logga in. Kanske hade fel syntax.

![Screenshot](img/Pasted%20image%2020260425114141.png)

Laddade om sidan, och det verkar som att jag lyckades! Tänk vilken stor skillnad det kan vara på `'` och `;` 

![Screenshot](img/Pasted%20image%2020260425114223.png)

Försöker logga in till adminpanelen nu med `superadmin@injectics.thm:superSecurePasswd101` och:

<details>
  <summary><b>Klicka här för att se flaggan</b></summary>

  ![Screenshot](img/Pasted%20image%2020260425114341.png)
</details><br>

Första flaggan i hamn!
## Flagga två
Nästa flagga ska hittas i mappen `flags`. Adminpanelen presenterar en ny funktion - en sida för att uppdatera sin profil och jag är övertygad om att det är vägen in.

![Screenshot](img/Pasted%20image%2020260425115136.png)

Skickar iväg en uppdatering för att fånga POST requesten. In med den till sqlmap och göra en körning. Nu med level 5 direkt.

Okej, level 5 var kanske lite overkill för det tar sjukt lång tid. Kört 10 min på första parametern utan träff, så skippar vidare till nästa. 

Hehe, tänkte kolla källkoden medan sqlmap körde. Möttes av det här :D

![Screenshot](img/Pasted%20image%2020260425182910.png)

Så, det känns som att fälten för för- och efternamn åtminstone är sårbara för nåt. Ska bara landa i exakt vad.

Den tuggar på. Parallellt med att den jobbar testar jag flaggan `--os-shell` från samma entry point vid ledartavlan - kanske att jag inte behöver hitta en ny ingång. Detta med `sqlmap -r leaderboard.txt -p gold --level=5 --risk=3 --os-shell --batch`

(Testade först utan level och risk eftersom jag redan scannat på djupet, men då lirade det inte)

Att ordna ett shell från ledartavlan funkar till synes inte.

![Screenshot](img/Pasted%20image%2020260425185639.png)

Låtit sqlmap köra länge nu, så nu blir det manuella tester.

Att lägga till en `'` i varje fält blir steg 1. 

Ingen påverkan på first name

![Screenshot](img/Pasted%20image%2020260425202154.png)

Inte heller vid email.

Det slog mig att modulen även tagit upp template injection - inte konstigt att sqlmap inte hittade några sårbarheter.

Jag testade att skicka `{{7*7}}` som förnamn och fick då 49 till svar!

![Screenshot](img/Pasted%20image%2020260425205438.png)
## Template injection

Dags att ta reda på vilket ramverk som körs då.

`{{7*'7'}}` gav också 49, så det skulle kunna vara Twig. (Hade det varit Jinja2 hade outputen varit `777777`. 

Nu blir det att titta i modulen, för syntaxen här blir riktigt krånglig...

Testade `{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}` men det gav:

![Screenshot](img/Pasted%20image%2020260425210535.png)

Det var inte det lättaste att hitta vad som är tillåtet - det är ganska nedlåst...

Har tagit hjälp av Claude och testat en uppsjö av olika kommandon, men antingen är de *not allowed* eller *unknown*. Det gäller verkligen att hitta exakt rätt angreppssätt.
## sstimap

Testar att installera **sstimap** för att se om det dels kan bekräfta vilken template engine som körs (visserligen övertygad om att det är Twig), dels hitta rätt exploit.

Efter ett par försök landade jag i följande kommando:

`sstimap -u "http://$IP/update_profile.php" -m POST -d "email=superadmin%40injectics.thm&fname=*&lname=hej" --cookie "0me0adiqgbfmvtjfknl3gnk6lp" --engine Twig`

Dock sa sstimap att formuläret inte är sårbart. Skumt. Fast inte så skumt egentligen, eftersom den renderade outputen inte syns på samma sida som formuläret. Formuläret finns på `/update_profile.php`, medan välkomstmeddelandet som syns på `/dashboard.php`. Detta är Second-Order SSTI, och efter att ha kollat i hjälpmenyn och sökt på nätet verkar sttimap inte ha något vettigt stöd för detta. Tyvärr...

Hittade ett önskemål (och förslag på hur man skulle kunna implementera det) på github: https://github.com/vladko312/SSTImap/issues/12

`{{attribute(_self, 'getEnvironment')}}` gav ändå nåt spännande, men inte så hjälpsamt om jag fattar det rätt.

![Screenshot](img/Pasted%20image%2020260426143614.png)

Hittar ingen väg in - inget är ju tillåtet!

![Screenshot](img/Pasted%20image%2020260426143739.png)

![Screenshot](img/Pasted%20image%2020260426143719.png)

![Screenshot](img/Pasted%20image%2020260426143814.png)

![Screenshot](img/Pasted%20image%2020260426143840.png)

Hittade dock en väldigt bra lathund på [HackTricks](https://hacktricks.wiki/en/pentesting-web/ssti-server-side-template-injection/index.html) för att kolla vilken template engine som används:

![Screenshot](img/Pasted%20image%2020260426144512.png)

![Screenshot](img/Pasted%20image%2020260426165028.png)
## Genombrottet

Ett litet genombrott med `{{['id','']|sort('passthru')}}` för den skriver faktiskt ut vilken användare vi kör som:

![Screenshot](img/Pasted%20image%2020260426163236.png)

Bytte `id` mot `ls` och fick följande:

![Screenshot](img/Pasted%20image%2020260426163435.png)

`ls flags` (körde först `ls -la /flags` men det visade ingenting, tog bort snedstrecket sen)

![Screenshot](img/Pasted%20image%2020260426163522.png)

`cat flags/5d8af1dc14503c7e4bdc8e51a3469f48.txt`

<details>
  <summary><b>Klicka här för att se flaggan</b></summary>

  ![Screenshot](img/Pasted%20image%2020260426163625.png)
</details>

Flagga två i hamn!

Jag är i princip 100% säker på att jag testade `passthru` igår, men säkert att jag gjorde samma misstag då som jag råkade göra till en början idag. Jag skrev `passthrough` eftersom *through* vanligtvis stavas så och jag har lagt ner mycket energi och möda på att lära mig den kluriga stavningen..! Well. Ännu en gång, ett typo kan vara det enda som står emellan en själv och flaggan :D Happy hacking!