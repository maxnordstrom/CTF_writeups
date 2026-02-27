# Mr. Phisher

https://tryhackme.com/room/mrphisher

![Screenshot](img/Pasted%20image%2020260226133755.png)

![Screenshot](img/Pasted%20image%2020260226133817.png)

När vi snurrat upp maskinen finns två filer

![Screenshot](img/Pasted%20image%2020260226133852.png)

I instruktionen för uppgiften står det "It keeps on asking me to "enable macros". What are those?" så jag antar att det är när man öppnar dokumentet som den frågan dyker upp.

Korrekt, fick upp den här varningen:

![Screenshot](img/Pasted%20image%2020260226134045.png)

Dokumentet innehåller en bild

![Screenshot](img/Pasted%20image%2020260226134116.png)

Dokumentet ska ju innehålla macron och dessa kan man hitta i LibreOffice genom **Tools > Macros > Edit Macros**

![Screenshot](img/Pasted%20image%2020260226134946.png)

Nu gäller det bara att lista ut hur koden fungerar! Men först, se om Libre kan köra macrot direkt. Kollar i **Options** och ser att jag kan ändra säkerhetsnivån. Dags att tillåta alla macron!

![Screenshot](img/Pasted%20image%2020260226144600.png)

Därefter **Run Macro**. Here goes nothing...

![Screenshot](img/Pasted%20image%2020260226145120.png)

Händer inget vad jag kan se i nuläget i alla fall.

Efter lite sökningar så verkar koden vara skriven i VBA (Visual Basics for Applications). Inte ett språk jag kan flytande, så ska försöka hitta en lathund.

Jag har förstått att LibreOffice använder sig av språket Basic, men att med hjälp av `Option VBASupport 1` så kan viss VBA användas. `Dim` deklarerar en variabel, och arrayen med alla siffror är antagligen decimalformen av olika tecken (ingen siffra är över 127).

Låter CyberChef decoda arrayen bara för att ta mig en titt.

Det blir dock lite tokigt eftersom 127 är med (DEL) och ett gäng siffror under 32 vilka inte är printable ASCII characters.

![Screenshot](img/Pasted%20image%2020260226151519.png)

Koden innehåller ju även `Xor`, så siffrorna i arrayen ska sannolikt inte decode:as rakt av. Efter lite mer sökningar tror jag att jag förstår koden.

```python
Rem Attribute VBA_ModuleType=VBAModule # Rem är en kommentar
Option VBASupport 1 # Ger stöd för VBA
Sub Format()
Dim a() # Deklarerar variabeln a som en array
Dim b As String # Deklarerar variabeln b som en string

# Här läggs arrayen till variabeln a
a = Array(102, 109, 99, 100, 127, 100, 53, 62, 105, 57, 61, 106, 62, 62, 55, 110, 113, 114, 118, 39, 36, 118, 47, 35, 32, 125, 34, 46, 46, 124, 43, 124, 25, 71, 26, 71, 21, 88)

# En for loop där i börjar på 0 och iterarer över arrayen a
For i = 0 To UBound(a)

# I loopen så läggs någonting till variablen b.
# Detta någonting är ett ASCII-tecken (som Chr tar 
# fram baserat på en siffra)
# Denna siffra tas från arrayen a vid index i, och denna siffra
# körs sen med Xor genom i.
b = b & Chr(a(i) Xor i)

# Next gör att loopen går om från början
Next

# När loopen är klar är processen slut
End Sub
```

Jag tänker att jag borde kunna lägga till en rad som skriver ut `b`. Verkar ju onödigt för mig att skriva om koden i nåt annat språk... En googling ger mig:

> **Output to the Immediate Window (Debug.Print)**: Use `Debug.Print` to display the value during debugging.

Får dock ett felmeddelande när jag lägger till det...

![Screenshot](img/Pasted%20image%2020260226153521.png)

Testar det andra förslaget, `MsgBox b` som ska skriva ut variablen `b` i en popup. Det lirade bra!

<details>
  <summary><b>Klicka här för att se flaggan</b></summary>

  ![Screenshot](img/Pasted%20image%2020260226153623.png)
</details><br>

Och det var den rätta flaggan!