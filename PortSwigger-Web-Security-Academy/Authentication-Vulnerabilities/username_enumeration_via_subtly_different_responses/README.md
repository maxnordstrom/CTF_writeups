# Username enumeration via subtly different responses

![Screenshot](img/Pasted%20image%2020260211115039.png)

Eftersom uppgiften vill att jag ska leta efter små skillnader i responsen så kommer jag att använda mig av samma tillvägagångssätt som tidigare - fånga ett inloggningsförsök, skicka det till Intruder. Köra bruteforce på användarnamnen (baserat på den givna listan) och hoppas att jag hittar något som säger att lösenordet är fel (istället för att användarnamnet är fel). Därefter brute force på lösenorden för att hitta rätt.

Felmeddelandet jag får för varje request i Intruder är `Invalid username or password`. Jag kan alltså inte avgöra något baserat på felmeddelandet. 

Responstiden skiljer sig dock. Användaren `admin` och `carlos` har en betydligt kortare responstid - kanske ett tecken på att de snabbt hittades i listan med användarnamn? 

De andra användarnamnen har en längre tid - ev. ett tecken på att servern har behövt gå igenom *hela* listan, för att till slut komma fram till att användarnamnet inte finns.

Användaren `af` har också en något kortare svarstid, men inte lika kort. Även `arlington`.

Mina social engineering-skills säger även att `carlos` bör finnas, då den användaren har funnits med i de tidigare labbarna.

Testar en **cluster bomb attack** med `admin` och `carlos` som användarnamn, och lösenordslistan för ja, lösenordet.

Hm, ingen av de två verkar ha resulterat i en lyckad inloggning. Vissa responstider är visserligen betydligt kortare än genomsnittet, och en är dubbelt så stor som genomsnittet, men alla får felmeddelandet `Invalid username or password`. Alla har även statuskod 200 - en lyckad inloggning skulle ev. kunna ha nån kod för redirect.

Testar samma sak fast med användarnamnen `af` och `arlington`.

Misstänker att jag inte kommer få nåt vidare resultat här heller. 

## Ta det från början

Läser instruktionen igen.

> To solve the lab, enumerate a valid username, brute-force this user's password, then access their account page.

Kommer ihåg att man fick ett tips tidigare i modulen - att ange ett väldigt långt lösenord. Så när man väl får träff på ett giltigt användarnamn kommer det ta tid för servern att processa det långa lösenordet. Ska testa det istället.

![Screenshot](img/Pasted%20image%2020260211201749.png)

Hm, det är fortfarande `af` som har längst tid med 192, därefter `apache` med 166. Testar `af`.

Ingen lycka. Jag testar igen med ett väldigt, väldigt långt lösenord istället...

![Screenshot](img/Pasted%20image%2020260211202905.png)

Med det långa lösenordet har `administrator` kortast responstid, medan följande fyra har längst:

![Screenshot](img/Pasted%20image%2020260211203408.png)

Jag testar först `at`. Sen en cluster bomb med alla.

Inget lyckat resultat med `at` heller. Nya tag imorgon...

## Ny dag

Jag lade märke till att ett antal responser innehåller en tom html-kommentar, typ en fjärdedel av responserna. Minns att de nämnde just det i modulen, att vissa responser kan skilja sig något. Att de är väldigt lika ett vanligt felmeddelande, men just att de skiljer sig på nåt ställe indikerar att det är en annan sida som laddas in, och då laddas den säker in av en anledning. Kanske för att användarnamnet är giltigt.

Jag ska plocka ut alla användarnamn som returnerar html-kommentaren och testa att brute force:a dem.

Det jobbiga är att det är 57 användarnamn som får den tomma html-kommentaren, 44 som inte har den. Det är ett för stort antal för att göra en effektiv brute force...

Okej, nu fick jag en snilleblixt. Äntligen. Jag skickade in ett användarnamn som uppenbart inte finns och kollade på responsen - den innehåller den tomma html-kommentaren. Alltså, de med tom html-kommentar i responsen är användarnamn som inte finns. Det är min teori.

Fast vad tusan... Nu när jag gör några stickprov så verkar det helt random... De som fick kommentar innan får det inte nu och tvärt om.

Har nu testat att skicka ett request till Repeater, och det är verkligen random när responsen innehåller den tomma html-kommentaren och inte... Det är alltså inte det som särskiljer ett request med giltigt användarnamn. Ska definitivt sova på saken...

## Ny dag igen

Idag, av någon anledning, så sticker responstiden för användaren `test` ut rejält:

![Screenshot](img/Pasted%20image%2020260219111723.png)

Responstiden på andraplats ligger på 99. 

Efter att ha gått igenom idéerna som presenterats i modulen gav jag mig ut på jakt efter ett typo, och det blev att jag chansade på att det skulle finnas ett typo i felmeddelandet, och det gjorde det! För användaren `ae` saknas en punkt i slutet av felmeddelandet:

![Screenshot](img/Pasted%20image%2020260219135357.png)

Alla andra responser innehåller felmeddelandet `Invalid username or password.` - alltså med en punkt på slutet.

Jag upptäckte detta genom manuellt arbete, genom att klicka mig igenom alla responser och scrolla ner till felmeddelandet. Jag kunde inte hitta något annat, smidigt sätt att göra detta på i Burp Community Edition.

Maskinen hann dock stänga ner innan jag hann testa lösenorden, och vid en omstart var användarnamnet `ae` inte giltigt längre - de verkar rotera giltiga användare. Men teorin om att den saknade punkten stämde fortfarande, för jag såg samma mönster för användaren `adkit` efter omstart. När jag hittade det giltiga användarnamnet var det bara att köra en password brute force och logga in.

![Screenshot](img/Pasted%20image%2020260219140942.png)
