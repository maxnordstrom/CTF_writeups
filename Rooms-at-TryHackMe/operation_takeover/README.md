![Screenshot](img/Pasted%20image%2020260410103247.png)

https://tryhackme.com/room/operationtakeover

![Screenshot](img/Pasted%20image%2020260410103308.png)

Spännande med ett rum utan instruktioner! Får se vad vi ställs inför.

## Initial recon

Vänder mig till nmap direkt för att se vad vi har att jobba med. Den här gången ska jag scanna *alla* portar :)

Hittar tre öppna portar:

![Screenshot](img/Pasted%20image%2020260410105542.png)

Kör en mer noggrann scanning på de tre portarna för att få fram vad som körs genom `nmap -sC -sV -p22,179,2623 $IP -v` och fick följande:

![Screenshot](img/Pasted%20image%2020260410111256.png)

Rummet handar ju om att hacka en router, så FRRouting på port 2623 är ju högst intressant. Jag blir även nyfiken på vad som gömmer sig bakom port 179 som endast returnerar `tcpwrapped`.

## FRRouting v 10.0

Gjorde en snabb googling om det finns några CVE för FRRouting version 10, och det verkar så

![Screenshot](img/Pasted%20image%2020260410105848.png)

Hittar ett par olika, de verkar dock alla handla om DoS, och vi vill ju ha access, inte sänka den...

Om jag testar att ansluta till routern med netcat så får jag en password promt - vi kan alltså ansluta till den. Sannolikheten att ett standardlösenord används? Vet ej. Vi får väl prova.

![Screenshot](img/Pasted%20image%2020260410111751.png)

En googling säger att FRRouting inte har något standardlösenord. Trist. Dra igång Hydra som tuggar på i bakgrunden? Ja, let's go. Rockyou får det bli.

## Hydra

Drog igång med kommandot `hydra -l "" -P /usr/share/wordlists/rockyou.txt telnet://$IP:2623`. 

FRRouting använder tydligen i regel inget användarnamn, därav den tomma strängen.

Men nu känns det som att jag började i fel ände. Jag skapar en custom wordlist istället för att prova min lycka där.

```
zebra
frr
frrouting
admin
password
router
cisco
bgpd
ospfd
root
toor
raspberry
dietpi
test
uploader
password
admin
administrator
marketing
12345678
1234
12345
qwerty
webadmin
webmaster
maintenance
techsupport
letmein
logon
Passw@rd
alpine
```

Dock ingen lycka där heller. Körde Rockyou ett tag men det känns lite väl långsökt att bara köra på den. Det måste finnas ett smartare sätt...

## BGP på port 179

Att nmap endast visar att port 179 kör bgp, men att det kommer tillbaka som tcpwrapped, är tydligen förväntat. Routern ansluter endast till IP:n som den har i sin lista.

Verkar dock klurigt att lista ut vilka IP:n som skulle finnas i den listan, så tittar vidare...

## CVE igen

Det har publicerats en ny CVE nyligen (mars 2026) som faktiskt handlar om access control.

![Screenshot](img/Pasted%20image%2020260410120128.png)

Jag gillar dock inte formuleringarna "The attack is considered to have high complexity" och "The exploitability is reported as difficult" :D

## Nmap igen

Nu kom jag ihåg att scanna alla portar. Dock endast TCP-portarna. Det kan nämligen finnas en läckande tjänst, SNMP, som använder sig av UDP.

Drog igång `nmap -sU -p- $IP -v` Det tog en väldans tid så jag stängde av just den.

Men eftersom vi är nyfikna på just SNMP på port 161 så kan jag använda programmet `onesixtyone`.

Detta programmet kör jag med en wordlist från seclists för att fuzza olika namn och fick följande träff:

![Screenshot](img/Pasted%20image%2020260410121355.png)

Därefter använder jag `snmpwalk` för att ta reda på mer info om routern. Kanske att den läcker lite götta.

`snmpwalk -v2c -c pr1v4t3 $IP | tee snmpwalk.txt` ger en himla massa information. Frågan är vad vi ska använda oss av.

Det visade sig att vi med hjälp av snmp kan köra kommandon på servern. Onind00 tog fram ett proof of concept genom att skriva en fil till servern och sedan läsa den:

![Screenshot](img/Pasted%20image%2020260410123618.png)

Därefter kunde vi skriva kommandon till servern med `snmpset` som vi sedan körde med `snmpwalk`

![Screenshot](img/Pasted%20image%2020260410125107.png)

![Screenshot](img/Pasted%20image%2020260410125131.png)

Den sista filen där ser ju lovande ut :)

![Screenshot](img/Pasted%20image%2020260410125158.png)

Vi fick flaggan!

![Screenshot](img/Pasted%20image%2020260410125214.png)

## Slutord

Det här var helt new grounds för mig. Körde den gamla hederliga kartläggningen från början, som visserligen lade grunden för att hitta de öppna portarna och tjänsterna som kördes, men själva attackkedjan hade jag ingen aning om. Så, tack onind00, och tack AI :D