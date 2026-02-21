![Screenshot](img/Pasted%20image%2020260130201133.png)

Min tanke är att skicka POST-requesten när jag laddar upp shell.php till Repeater så att jag kan modifiera filnamnet och testa mig fram.

Okej, jag kan bara ladda upp jpg och png

![Screenshot](img/Pasted%20image%2020260130201427.png)

Med requestet i Repeater börjar jag med att ändra headern `Content-Type:` till `image/png` (istället för application/x-php)

Testar att döpa filen till `shell.php.png`. Fick detta till svar:

![Screenshot](img/Pasted%20image%2020260130201826.png)

Och den verkar vara uppladdad som min avatar:

![Screenshot](img/Pasted%20image%2020260130201902.png)

Vad händer om jag försöker komma åt den? Då kom jag bara till bilden:

![Screenshot](img/Pasted%20image%2020260130201954.png)

Om jag kör ett GET-request i Burp då? Då såg jag filens innehåll:

![Screenshot](img/Pasted%20image%2020260130202055.png)

Kanske att jag ska låta `Content-Type` vara `application` trots allt? Testar med det. Det gick att ladda upp:

![Screenshot](img/Pasted%20image%2020260130202209.png)

I webbläsaren såg jag samma sak som innan, och även i Burp. I responsen i Burp kunde jag se att Content-Type var ändrag till image/png, så det måste ha skett server-side.

Testar att köra en null-byte innan .png. Alltså `shell3.php%00.png`. Då fick jag det här svaret

![Screenshot](img/Pasted%20image%2020260130202706.png)

Verkar som att den har laddat upp en renodlad php-fil den här gången. Om jag kör ett GET-request i Burp? Då fick jag flaggan.

![Screenshot](img/Pasted%20image%2020260130202755.png)