# Fowsniff CTF

![Screenshot](img/Pasted%20image%2020260427091512.png)

https://tryhackme.com/room/ctf

![Screenshot](img/Pasted%20image%2020260427091647.png)

Låter som en bra uppvärmning för en tisdagsmorgon.
## Scan

![Screenshot](img/Pasted%20image%2020260427091808.png)

![Screenshot](img/Pasted%20image%2020260427091932.png)

Okidoki - antagligen en live hemsida och mailtjänster.
## Hemsidan
På hemsidan står det bara att de blivit utsatta för en attack.

![Screenshot](img/Pasted%20image%2020260428082912.png)

## Mail

På hemsidan kan man läsa:

![Screenshot](img/Pasted%20image%2020260428083026.png)

Vore trevligt att hitta dessa credentials.

Hittade md5-hashar på en gammal pastebin via google -> twitter -> pastebin -> waybackmachine. Pastebin-sidan var nämligen borttagen p.g.a. dess innehåll, men Tobzon kom med den briljanta idén att kolla waybackmachine.

Körde crackstation på hasharna.

![Screenshot](img/Pasted%20image%2020260427093017.png)

Alla utom en, det får jag änså säga är ett lyckat resultat. Satte upp en wordlist med alla creds och körde Hydra mot pop3-servern.

![Screenshot](img/Pasted%20image%2020260427094650.png)

![Screenshot](img/Pasted%20image%2020260427094602.png)

Fanns två mail på servern.

![Screenshot](img/Pasted%20image%2020260427094758.png)

Trevligt med ett temporärt lösenord till ssh :)

![Screenshot](img/Pasted%20image%2020260427095030.png)

Det andra mailet förstår jag inte mycket av, jaja.

Körde Hydra mot ssh för att ta reda på vilken användare lösenordet fungerade på.

![Screenshot](img/Pasted%20image%2020260427095842.png)

![Screenshot](img/Pasted%20image%2020260427095827.png)

Inloggad!

![Screenshot](img/Pasted%20image%2020260427095925.png)

Lärde mig ett nytt kommando för enumeration: `getent group`

Sökte på filer som min nuvarande grupp kan skriva till:

`find / -writable -type f 2>/dev/null` 

Grepade ut txt-filer med en enkel pipe `| grep txt`

![Screenshot](img/Pasted%20image%2020260427101614.png)

Det visade sig att vi kan skriva till filen som innehåller ASCII-bannern som visas när man loggar in via ssh. Scriptet som körs vid inloggning hämtar textfilen, och scriptet körs som root.

Lade in ett reverse shell från pentestmonkey. Blev lite fel först för att jag hade kikat på en version som inte hade wrappat IP-numret med citattecken. Men, när det väl var på plats fick jag ett reverse shell när jag loggade in via ssh.

![Screenshot](img/Pasted%20image%2020260427103711.png)

Inga flaggor att hämta, bara en go känsla av att äga maskinen :)