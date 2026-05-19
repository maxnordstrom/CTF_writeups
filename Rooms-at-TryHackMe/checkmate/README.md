![Screenshot](img/Pasted%20image%2020260519094339.png)

https://tryhackme.com/room/checkmate
## The Task

> Marco Bianchi, a systems administrator, recently deployed several internal services, including a firewall console, employee portal, social platform, and SSH access to critical infrastructure. Due to tight deadlines and operational pressure, Marco reused weak, predictable, and pattern-based passwords across multiple systems.  

> Your objective is to conduct a password security assessment to identify weaknesses in Marco’s authentication practices.

![Screenshot](img/Pasted%20image%2020260519094509.png)
## Till hemsidan

Har fått ett IP, så hur ser applikationen ut? Jag kommer till en portal där jag kan uppge de knäckta lösenorden.

![Screenshot](img/Pasted%20image%2020260519095759.png)
## Level 1

Första applikationen att ta en närmare titt på är en brandvägg som finns på port 5001.

![Screenshot](img/Pasted%20image%2020260519095939.png)

Testade med `admin:admin`, men det lirade inte. 

Ser att domännamnet `firewall.thm` är utskrivet i rutan. Kanske lägga till den i hosts-filen och fuzza lite? Det gav inget av värde

![Screenshot](img/Pasted%20image%2020260519100835.png)

Inget som avslöjar vilken typ av brandvägg det är (märke, modell o.s.v.), så det blir svårt att googla standard-credentials. Drar igång en Hydra med vanliga lösenord.

`hydra -l admin -P /usr/share/seclists/Passwords/Default-Credentials/default-passwords.txt firewall.thm -s 5001 http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid" -I`

Och hittade rätt lösenord efter nån sekund

<details>
  <summary><b>Klicka här för att se lösenordet</b></summary>

  `12345`
</details>
  
## Level 2

![Screenshot](img/Pasted%20image%2020260519102211.png)

Ett inlogg för anställda på sajten `jobs.thm`. Lösenordet ska tydligen bygga på vanliga nyckelord som förekommer på sajten.

![Screenshot](img/Pasted%20image%2020260519102542.png)

Vi presenteras ju för ett gäng keywords redan vid rubriken:

![Screenshot](img/Pasted%20image%2020260519102634.png)

När man trycker på Employee Login kommer vi till inloggningsformuläret:

![Screenshot](img/Pasted%20image%2020260519102747.png)

Tack för det förifyllda användarnamnet!

Drog igång en attack i Burp Intruder och fick träff!

<details>
  <summary><b>Klicka här för att se lösenordet</b></summary>

  ![Screenshot](img/Pasted%20image%2020260519103051.png)
</details>


När man loggat in ser det ut så här:

![Screenshot](img/Pasted%20image%2020260519103340.png)

Här hittar vi lite info som kan komma till nytta senare - efternamn, smeknamn och födelsedag.
## Level 3

![Screenshot](img/Pasted%20image%2020260519103456.png)

![Screenshot](img/Pasted%20image%2020260519103613.png)

![Screenshot](img/Pasted%20image%2020260519105146.png)

Testar att använda `marky` som username och några av de tidigare lösenorden. Inga träffar. Körde rockyou utan träff, och skapade även en custom wordlist med Claude utan att lyckas. Testade att skapa en custom wordlist med **cupp** som ordnade en lista på ca 2500 ord. Det blev ingen träff då heller. Ändrade username till `marco` (som det var på jobbsajten) och då blev det träff!

<details>
  <summary><b>Klicka här för att se lösenordet</b></summary>

  `Bianchi2495`
</details>

Lite lurig fint där med nicknamet, och det var inte ens med i lösenordet. Kul! Lite märkligt bara var `24` kommer in i bilden, födelsedatumet var ju **14021995** (enligt DDMMYYY). Jaja.
## Level 4

![Screenshot](img/Pasted%20image%2020260519110121.png)

![Screenshot](img/Pasted%20image%2020260519110219.png)

Högerklickade på profilbilden för att kolla filnamnet:

![Screenshot](img/Pasted%20image%2020260519110329.png)

Innan jag vänder mig till Hashcat får det bli Crackstation. Och det gav resultat:

<details>
  <summary><b>Klicka här för att se lösenordet</b></summary>

  ![Screenshot](img/Pasted%20image%2020260519110537.png)
</details>

## Level 5

![Screenshot](img/Pasted%20image%2020260519110632.png)

![Screenshot](img/Pasted%20image%2020260519110906.png)

Skapade en ny wordlist med **cupp**, den landade på ca 10500 ord. Blev inte helt nöjd med listan för den skapade så många ord som var utanför Marcos rekommendation. Blev att jag vände mig till Claude och skräddarsydde en wordlist helt enligt Marcos regler. Den blev på ca 500 ord och Hydra fick träff mer eller mindre omedelbart.

<details>
  <summary><b>Klicka här för att se lösenordet</b></summary>

  ![Screenshot](img/Pasted%20image%2020260519113218.png)
</details>

Det var det, happy hacking!