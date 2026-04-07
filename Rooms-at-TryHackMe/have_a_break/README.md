# Have a Break

![[Pasted image 20260407165055.png]]

https://tryhackme.com/room/haveabreak

![[Pasted image 20260407165129.png]]

## Fråga 1

![[Pasted image 20260407165626.png]]

Man ska ta reda på vilken VPN-tjänst som har använts för att skicka det anonyma mailet. Jag kollar mailets headers och slår upp de IP-nummer jag kan hitta.

Det första IP-numret leder till Google.

Det andra till 31173 Services AB som är en VPN-tjänst, men jag hittar inget namn som stämmer överens med vad de söker i svaret.

![[Pasted image 20260406152722.png]]

Får leta vidare.

Hittar denna men det är en privat adress:

![[Pasted image 20260406152938.png]]

Gick tilbaka till adressen som faktiskt var en VPN-tjänst. Gjorde en sökning på `whois.domaintools.com` (hade ju lika bra kunnat köra `whois` i terminalen förvisso...) och såg att företaget var registrerat på en svensk adress:

![[Pasted image 20260406153710.png]]

Hur många svenska VPN-tjänster finns det med ett namn på 7 tecken? Testade `Mullvad` och det var rätt. Undra hur man skulle kunna hitta uppgifter som faktiskt bekräftar att det är så, men men.

Jag sökte vidare i jakt på att få det helt bekräftat. Hittade en sajt som heter `ipinfo.io`, och om man skapade ett konto och gick till `ipinfo.io/IP` fick man info om vilken VPN-tjänst som användes. Bra sajt att lägga på minnet!

![[Pasted image 20260407091435.png]]

## Fråga 2

![[Pasted image 20260407091607.png]]

Vi har följande bild att utfå från:

![[Pasted image 20260407101117.png]]

Efter många sökningar på google maps kombinerade jag en bildsökning med `overnight parking` och fick följande:

![[Pasted image 20260407100702.png]]

Verkar dock inte vara den heller, den ligger för nära Brno.

Titta lite mer i dokumentet vi fått som beskriver bilden:

![[Pasted image 20260407101246.png]]

Bilden ska vara tagen i närheten av Hulín... Tillbaks till google maps. Fast när jag läste om det så står det ju att bilden *beslagtogs* i närheten av Hulín - bilden behöver alltså inte vara tagen där...
## Fråga 3

![[Pasted image 20260407101859.png]]

Har fått en access log, tittar i den.

Vi har även fått tillgång till ett mail där det står att någon har sett misstänkt aktivitet natten innan avresa.

![[Pasted image 20260407102119.png]]

Här är de två sista händelserna i loggen som skulle kunna vara misstänkta...

![[Pasted image 20260407102257.png]]

22:14:09 var rätt svar på den.
## Fråga 4

![[Pasted image 20260407102351.png]]

Avsändaren av mailet är `notmyname2847@gmail.com`

Vi har fått en personallista att utgå från:

![[Pasted image 20260407102540.png]]

Det finns ingen koppling mellan mailadressen och något i personallistan.

Vi kan se att mailet skickades från en Android-telefon:

![[Pasted image 20260407102814.png]]

Men att döma av mailet så är det någon som såg misstänkt aktivitet i de interna systemen, och någon som ser sådan aktivitet bör vara någon inom IT? Värt att testa.

Det var inte rätt... Man skulle kunna brute force:a det hela, inte så många i personalen ju, men det är ju inte rätt sätt att lösa det på. 
## Fråga 5

![[Pasted image 20260407103202.png]]

Det borde vara personen som gjorde exporten av route-dokumentet. Och det var det!

## Fråga 6

![[Pasted image 20260407104203.png]]

Jag trodde att det skulle vara lika enkelt som att kolla i personallistan, men där står ju inga namn. 

I en rapport över kommunikation står det att en extern emailadress har begärt åtkomst till en fil, men blivit blockerad. Adressen är `kraliknovak09@gmail.com`, men en google-sökning på det ger inte mycket. Inte mer än att över 280 sökningar har gjorts på adressen de senaste 7 dagarna :D

Hittade en annan sajt för att kolla upp epost-adresser, `epieos.com`

![[Pasted image 20260407130531.png]]

Vi får se om den lever upp till namnet.

Efter registrering gjorde jag en sökning och fick följande

![[Pasted image 20260407131248.png]]

Där kan vi se att personen i fråga var "busy" mellan 22 och 00 den 26:e mars

![[Pasted image 20260407131413.png]]

Och när jag följde länken till Google Maps verkar den peka till en lokal guide

![[Pasted image 20260407131549.png]]

Frågan är nu om detta är namnet på läckan, eller personen som skickade det anonyma mailet?

Bytte flik till Anmeldelser/recensioner och hittade ett betyg för en viss bensinmack...

![[Pasted image 20260407131840.png]]

Adressen för den bensinmacken var rätt svar på **fråga 3**. Yey!

Hur är det med Radovan då, kan det vara svaret på sista frågan? Japp, det var rätt svar på **fråga 6.** Super!

Nu till fråga 4, vilken av de anställda som skickade det anonyma mailet...

Mailet skickades ju även den från en gmail-adress, närmare bestämt `notmyname2847@gmail.com`. Kan ju hända att det går att hitta nåt spännande.

Det gav tyvärr inga träffar

![[Pasted image 20260407134150.png]]

Då är det en riktigt klurig fråga...

Jag läser i den bifogade pdf-filen igen om vad som behöver göras.

![[Pasted image 20260407143119.png]]

Så, vi har ju titta på några headers i det bifogade mailet - kanske att det finns ytterligare metadata som avslöjar ett personal-ID? Hittar dock ingen ytterligare info som kan tänkas vara användbar...

Nu vänder vi oss till de lite mer obskyra metoderna. Skickat ett mail till adressen i hopp om ett avslöjande autosvar. Ingen lycka. Testade att logga in på kontot. Ingen lycka, kontot verkar inte finnas...

![[Pasted image 20260407152254.png]]

Nu gjorde jag en chansning där min teori är att det är personen som loggar in på systemet direkt efter exporten som reagerar på att någon har varit inne vid en anmärkningsvärd tid.

![[Pasted image 20260407162326.png]]

Det är alltså en Dispatch Operator som loggar in, märker det mystiska och därefter mailar lokaltidningen.

Känner mig lite snuvad på det logiska just på denna fråga, för det skulle ju i teorin kunna vara vilken anställd som helst som loggar in i systemen efter att exporten har skett. Ska bli intressant att läsa fler writeups på denna.