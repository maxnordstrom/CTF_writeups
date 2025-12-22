# WARMUP: /files (Misc)

![[Pasted image 20251215232724.png]]

En uppvärmnings-utmaning i kategorin Misc som lämnar mycket åt fantasin - det finns ingen beskrivning. Det man har att gå på är titeln, en bild på Inspector Gadget och tema-låten till densamme.

Mina lagkamrater hade redan tittat på uppgiften och då analyserat både bild och ljudfil utan att hitta något direkt intressant. En i laget nämde att det kom ett ljud i musiken som stack ut lite och att det kanske innehöll något av värde. Jag hade en annan approach.

### Utmaningens titel

Eftersom utmaningen hette **/files** tänkte jag att det kanske fanns en katalog på servern med det namnet, och att man på något sätt skulle kunna hitta andra intressanta filer där, till exempel en flagga.

Jag kollade därför i inspector för att se var ifrån bilden hämtades.

![[Pasted image 20251215231931.png]]

I efterhand så kan jag ju se att jag hade haft första halvan av flaggan redan där, men min syn var helt inställd på att leta efter **/files**, och det hittade jag ju där!

Jag öppnade länken i en ny tab som dels laddade ner bilden och visade den, dock från min lokala katalog, men det öppnades även ytterligare en tab: `https://ctf.samurai.nu/files/Th1nk1ng_Out51/inspector-gadget.webp`

Jag försökte navigera till `https://ctf.samurai.nu/files/` men det resulterade i 404.

Sen tittade jag på länken och, aha, jag ser något som skulle kunna vara en flagga! Jag sparade strängen `Th1nk1ng_Out51`, gick tillbaka till uppgiften och gjorde samma sak med ljudfilen, som av en händelse fanns på `/files/d3_Th3_Ch4ll3ng3_B0x/Inspector_Gadget_Theme.mp3`. Där var andra halvan. 

Jag wrappade båda halvorna i CTF:ens flaggformat och lämnade in. Det var rätt!

<details>
  <summary><b>Klicka för att se flaggan</b></summary>
  
  `O24{Th1nk1ng_Out51d3_Th3_Ch4ll3ng3_B0x}`
</details> 