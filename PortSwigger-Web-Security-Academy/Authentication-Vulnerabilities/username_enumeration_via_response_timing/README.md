# Username enumeration via response timing

![Screenshot](img/Pasted%20image%2020260219141542.png)

Eftersom jag har fungerande credentials planerar jag att försöka logga in med det giltiga användarnamnet men med ett felaktigt lösenord för att se hur lång tid det tar för servern att svara. Därefter med ett uppenbart felaktigt användarnamn för att se om servern tar längre eller kortare tid på sig att svara.

När jag har koll på svarstiderna kör jag en username enumeration och letar efter en svarstid som matchar den tid jag fick för användarnamnet `wiener`.

Här är mina försök med användaren `wiener`, både med ogiltiga och det giltiga lösenordet:

![Screenshot](img/Pasted%20image%2020260311151950.png)

Nu använder jag samma lista fast med ett användarnamn som är ogiltigt, det landade på användarnamnet `TheKingOfKingsAndQueensAndPrinces`. Här är resultatet:

![Screenshot](img/Pasted%20image%2020260311152221.png)

Ingen *aha-upplevelse*, så jag får väl bränna igenom hela listan med användarnamn och se om det är något som sticker ut.

Vid en första genomkörning hade jag `password` som lösenord, men när jag går tillbaka och läser om texten från modulen står det så här:

![Screenshot](img/Pasted%20image%2020260311152741.png)

Jag kan alltså köra en brute force med användarnamnen, och skriva in ett onödigt långt lösenord som payload för att se om det ger en längre svarstid.

Med ett kort lösenord så blir svarstiderna rätt jämt spridda mellan 55 och 217, ingen som sticker ut jättemycket.

Testar med ett långt istället

![Screenshot](img/Pasted%20image%2020260311155351.png)

Då är majoriteten av svar runt 90, medan ett par klättrar en bit över 100. Jag ska testa en Cluster bomb attack med `administrator` och `application` tillsammans med listan över giltiga lösenord.

![Screenshot](img/Pasted%20image%2020260311160155.png)

Nu tittade jag lite närmare på responsen och lade märke till den här lilla skojiga grejen:

![Screenshot](img/Pasted%20image%2020260311160622.png)

Känns som att det går utanför labbens tema något, men jag behöver nog hitta ett sätt att undgå rate limit. Kanske är det det som står under hinten? :D Jag har inte kollat, men ska göra ett försök först.

Jag lade in headern `X-Forwarded-For:` (från grundkursen Bypass Rate Limit 101, haha) och ställde in en Pitchfork attack för att kringgå rate limit. Payload 1 genererade en siffra mellan 0 och 255 (för att simulera en giltig IP-adress), medan Payload 2 gick igenom användarnamnen.

![Screenshot](img/Pasted%20image%2020260311162723.png)

Jag misstänker att jag hittade rätt användarnamn ;)

![Screenshot](img/Pasted%20image%2020260311162816.png)

I och med att svartiden är såpass mycket längre kan man misstänka att det är det enda användarnamnet servern testade lösenordet mot.

Dags att köra brute force på användarnamnet `ftp`. Och jag behåller min X-Forwarded-For header.

Det blev en Redirect 302 med lösenordet `harley`

![Screenshot](img/Pasted%20image%2020260311163135.png)

Och jag lyckades logga in med `ftp:harley`

![Screenshot](img/Pasted%20image%2020260311163214.png)

Haha, jag kollade hinten nu, och det var som jag trodde :D

![Screenshot](img/Pasted%20image%2020260311163858.png)