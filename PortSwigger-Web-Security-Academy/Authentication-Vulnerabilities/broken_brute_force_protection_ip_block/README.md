# Lab: Broken brute-force protection, IP block

![Screenshot](img/Pasted%20image%2020260318092805.png)

Steg ett blir att försöka ta reda på vad för typ av blockering som implementeras och efter hur många misslyckade inloggningar. En central del som modulen tar upp är att IP-loggningen kan nollställas så fort man gör en lyckad inloggning. Genom att lägga in sina egna, giltiga användaruppgifter regelbundet i en brute-force attack skulle man alltså kunna kringgår rate limit.

Jag förstår i teorin hur jag ska gå till väga, frågan är hur man gör det i Burp. Jag börjar med att testa lite manuellt.

Gjorde ett inloggningsförsök med ett uppenbart felaktigt lösenord för att se om något i response headers avslöjar vad som händer. Dock stod det ingenting. Skickar vidare till repeater och upprepar.

Efter 4-5 felaktiga inloggningsförsök kom detta felmeddelande:

![Screenshot](img/Pasted%20image%2020260318093454.png)

Efter att ha gjort en lyckad inloggning med mina egna credentials och därefter gjorde ett nytt försök med `carlos` kom det ursprungliga felmeddelandet igen:

![Screenshot](img/Pasted%20image%2020260318093635.png)

Så - vi behöver konfigurera en attack med Intruder där vi slänger in våra egna credentials med jämna mellanrum för att förhindra att mitt IP blir blockerat.

Efter lite sökningar förstod jag att det enklaste sättet är att skapa två wordlists. Jag hade hoppats på att det skulle finnas nåt snyggt sätt att göra detta i Burp, men det får bli lite manuellt arbete innan vi kan kicka igång Intruder.

Jag skapade en lista med användarnamnet `carlos` 100 gånger (lika lång som min password list) genom `yes carlos | head -n 100 > usernames.txt`

Därefter insertade jag det giltiga användarnamnet `wiener` på var fjärde rad genom kommandot `sed '3~3 a\wiener' usernames.txt > usernames2.txt`

Därefter upprepade jag samma process för min password list, så att det giltiga lösenordet `peter` kom på var fjärde rad.

![Screenshot](img/Pasted%20image%2020260318100228.png)

![Screenshot](img/Pasted%20image%2020260318100329.png)

Då är det dags att köra Intruder.

Satte upp en pitchfork attack som jag trodde skulle funka

![Screenshot](img/Pasted%20image%2020260318101145.png)

Men, även om rätt credentials skickas iväg får vi samma felmeddelande om att vi har nått rate limit...

Endast vid två tillfällen lyckas inloggningen med mina giltiga credentials:

![Screenshot](img/Pasted%20image%2020260318101442.png)

Detta känns lite märkligt...

Det som känns mest sannolikt är väl att vi försöker med för många `carlos` innan det kommer ett giltigt `wiener`. Redigerar mina wordlists så att mina giltiga credentials kommer på var tredje rad istället.

Det blev till synes ingen skillnad...

Testar att inleda med mina korrekta credentials istället. Ändrade värdet för mina payload positions så att värde 0 är mina giltiga credentials istället för att endast vara en placeholder.

![Screenshot](img/Pasted%20image%2020260318102640.png)

Därefter lirade Intruder som jag tänkt:

![Screenshot](img/Pasted%20image%2020260318102713.png)

Var tredje request, med mina giltiga credentials, resulterar i en 302 redirect, super. Dags att leta efter en lyckad inloggning för `carlos`

Sorterade på statuskod och kunde snabbt se användarnamnet vi letade efter:

![Screenshot](img/Pasted%20image%2020260318102836.png)

Rätt är alltså `carlos:jordan`

Lyckad inloggning!

![Screenshot](img/Pasted%20image%2020260318102911.png)