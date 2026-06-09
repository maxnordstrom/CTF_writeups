# Cache Me Outside

*2026-06-08*

![Screenshot](img/Pasted%20image%2020260608093003.png)

https://tryhackme.com/room/cachemeoutside
## Intro

> Years after walking away from the scene, a retired hacker has left pieces of his identity scattered across the open internet.

> At first glance, it looks like nothing more than a leaked conversation screenshot. But buried in that image is the first thread of a much larger trail. Public profiles, forgotten details, and small mistakes begin to connect into something more deliberate.

> Someone wanted this person found.
## Uppgift

> You are an OSINT investigator tasked with identifying the retired hacker and tracing the clues he left behind.

> Start with the conversation screenshot, follow his online presence, connect the exposed details, and use the final evidence to determine where the trail ends.
## Flagga 1 (Full Name)

Följde länken som fanns i bilden:

![Screenshot](img/Pasted%20image%2020260608093046.png)

Då kom man till en social media (Komoot), och där stod namnet.
## Flagga 2 (Email)

På Komoot fanns en länk till github. Jag kollade på det enda repot och den enda commiten. Lade till `.patch` i URL:en och kunde läsa ut mailadressen.

<details>
  <summary><b>Klicka här för att se mailen</b></summary>

  ![Screenshot](img/Pasted%20image%2020260608093326.png)
</details>

## Flagga 3 (Phone)

Vi letade länge och väl utan att hitta - kollade sociala medier, google maps, calendar, epieos.com utan att lyckas. Tidigt i processen hade jag slängt ut idén om att skicka ett mail till mailadressen i hopp om ett autosvar med telefonnumret i, men det kändes långsökt. Till slut skickade **Tobzon** ett mail och fick ett instant autosvar :D
## Flagga 4 (City)

Hittade ett konto för användaren `jiml33t` på Threads. Där fanns en bild där ett hus hade `irigatii.ro` på fasaden. Sökte på google maps och hittade huvudkontoret. Stadens namn passade in på svaret.
## Flagga 5 (Tram Station)

Kollade efter en fransk supermarket i samma stad som ovan. Hittade dock ingen så utgick från samma ställe som huvudkontoret. Några hållplatser bort hittades det rätta svaret.