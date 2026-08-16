# Room 404

https://tryhackme.com/room/hh-room404-804573bf

![Screenshot](img/Pasted%20image%2020260809182218.png)

![Screenshot](img/Pasted%20image%2020260809182243.png)
## Intro
Ja, det finns inte så mycket att säga mer än vad som står i checklistan ovan. Jag har fått ett IP och tips om att port 8080 är öppen. En snabb titt med curl visar att det hostas en hemsida, inte helt oväntat. Let's go!
## Recon
Kollade sajten i webbläsaren, ser trevlig ut. Skickade iväg en directory enumeration med Gobuster därefter. Det var kanske allt som behövdes?

![Screenshot](img/Pasted%20image%2020260809192102.png)

Curl till den endpointen?

![Screenshot](img/Pasted%20image%2020260809192709.png)

Men att bara peka till `/.git` resulterade i:

![Screenshot](img/Pasted%20image%2020260809192741.png)

Tar det i webbläsaren.

Hm, vilken jag än klickar på resulterar i en liten 404

![Screenshot](img/Pasted%20image%2020260809192916.png)

Rummet heter ju trots allt `Room 404`...

Om jag skriver URL:en manuellt utan att klicka på länken så kommer jag rätt, och det går att navigera genom merparten av `/.git` både i webbläsaren och curl, men det är lite bökigt. Vore ju mest smidigt att kunna söka genom det lokalt.

`git clone` mot URL:en? Nope...

![Screenshot](img/Pasted%20image%2020260809205659.png)

Skapade istället ett virual environment och installerade `git-dumper` 

Med git-dumper kunde jag dra ner hela repot och arbeta i det lokalt

![Screenshot](img/Pasted%20image%2020260809211727.png)

Läste ut `README.md` och hittade flaggan!