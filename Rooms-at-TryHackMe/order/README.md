# Order

https://tryhackme.com/room/hfb1order

![Screenshot](img/Pasted%20image%2020260227145334.png)

![Screenshot](img/Pasted%20image%2020260227145347.png)

Kör en XOR brute force i CyberChef med den kända plaintexten, men det gav inte så mycket.

Visade sig sen att jag bara klistrat in halva meddelandet i CyberChef - jag trodde det var två olika. Nu klistrar jag in hela och får lite mer data att arbeta med åtminstone.

Vi har ju fått en del av plaintexten, men när vi använder den som key i en XOR operation blir det mestadels skräptecken.

Efter lite sökningar visade det sig att jag behövde decoda den krypterade texten från hex innan jag kör XOR.

Jag slängde in en **From Hex** innan XOR operationen och då får vi fram det som verkar vara nyckeln:

![Screenshot](img/Pasted%20image%2020260227152857.png)

Nu skriver jag in SNEAKY som nyckel istället och får då fram:

<details>
  <summary><b>Klicka här för att se flaggan</b></summary>

  ![Screenshot](img/Pasted%20image%2020260227152933.png)
</details><br>

Så, för att summera hur XOR funkar:

- plaintext XOR key = ciphertext (så man vanligtvis krypterar)
- ciphertext XOR plaintext = key
- key XOR ciphertext = plaintext

Och det var det vi gjorde ovan. Vi tog plaintext (ORDER:) och XOR:ade det med cipertext och fick ut key (SNEAKY). Därefter körde vi key XOR chipertext och fick ut plaintext. Och flaggan!