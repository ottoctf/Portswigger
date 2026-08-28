# Lab: Blind SQL injection with conditional errors

This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and the application does not respond any differently based on whether the query returns any rows. If the SQL query causes an error, then the application returns a custom error message.

The database contains a different table called users, with columns called username and password. You need to exploit the blind SQL injection vulnerability to find out the password of the administrator user.

To solve the lab, log in as the administrator user.

I denna writeup kommer jag inte ha en `Förklaring` sektion utan kommer förklara stegen under lösningen istället.

## Lösning

Första steget var att fånga en HTTP-request till `Home` i Burp Proxy och skicka den till Burp Repeater.

Man fick en del ledtrådar i beskrivningen, jag visste därför att injektionssårbarheten finns i `TrackingId`-cookien. 
Vanligtvis hade man behövt enumerera mer för att komma fram till den slutsatsen.

Eftersom detta är en blind SQL injection med conditional errors så började jag med att skriva en apostrof efter `TrackingId` för att observera vad jag får för HTTP-response.

<img width="1264" height="679" alt="image" src="https://github.com/user-attachments/assets/6131cc8f-a096-404b-a963-4989496c1d81" />


Man ser att man här får ett 500-svar vilket indikerar att apostrofen påverkar SQL-frågan och orsakar ett syntax fel.

Därefter provade jag då att skriva ännu en apostrof för att se om jag får annorlunda svar.

<img width="1261" height="677" alt="image" src="https://github.com/user-attachments/assets/963cc8de-7d90-4014-bfc2-5d1853d10a4f" />


Man ser här att jag får nu istället ett 200 svar. Samma sak gäller om jag skulle ha 4 apostrofer.

Nästa steg var att ställa en enkel SQL-fråga för att se om det bryter syntaxen eller inte. 

```sql
' ||(SELECT '')||'
```
Vad detta gör i praktiken är exakt samma som att skriva 4 apostrofer som jag gjorde tidigare, det borde därför också ge ett 200 svar.

<img width="1263" height="683" alt="image" src="https://github.com/user-attachments/assets/af01990b-9931-4bfc-9c01-27dd48ae94c6" />


Det gav ett 500-svar vilket egentligen inte borde hända, eftersom att jag inte fick veta vilket databasspråk det är i denna labb är det värt att testa om det kan bero på det.

Därför testar jag att lägga in `FROM dual` vilket är en dummy tabell som finns i Oracle. Den existerar eftersom Oracle kräver en `FROM` del och ifall man exempelvis 
behöver en giltig tabell att hämta data från så existerar `dual` för att man ska kunna uppfylla kravet.

<img width="1266" height="694" alt="image" src="https://github.com/user-attachments/assets/f87c6869-a4db-427c-aaea-f8be6243e557" />

Jag provar därefter för att vara säker ett tabellnamn som inte existerar.

<img width="1264" height="677" alt="image" src="https://github.com/user-attachments/assets/2ef4dbcc-c94c-4277-aef9-4383b4c9ad94" />

Eftersom detta gav ett 500-svar istället indikeras det att databasen använder Oracle, samt att det går att få olika svar beroende på om tabellnamnet existerar eller inte.

Jag fick veta i beskrivningen att tabellen `users` ska existera, därför testar jag att hämta `''` från `Users` och sedan observera svaret som jag får tillbaka.

<img width="1262" height="681" alt="image" src="https://github.com/user-attachments/assets/6d2321f6-22a6-46b9-a455-04099604bafc" />

Utan `ROWNUM=1` kan subqueryn returnera en rad för varje rad som finns i `users`. Eftersom sättet jag använder subqueryn på så behöver den returnera högst en rad. Genom `WHERE ROWNUM=1` begränsar jag resultatet till en rad vilket då inte orsakar en krasch i subqueryn.

<img width="1264" height="678" alt="image" src="https://github.com/user-attachments/assets/ad9e3fb3-17e2-49d3-b0cb-e70fc66ba9cf" />

Detta gav tillbaka ett 200 svar vilket indikerar att tabellen `users` existerar.

Jag valde därefter att verifiera om användaren `administrator` faktiskt existerade.

För det använder jag `TO_CHAR` funktionen som används för att konvertera ett värde till text. 

Om jag har SQL-frågan:
```sql
SELECT TO_CHAR(1/0) FROM users WHERE username='administrator'
```
Då kommer SQL leta upp administrator. Ifall administrator då inte existerar så kommer inte värdet i `TO_CHAR` beräknas, alltså 1/0 kommer inte att utföras och jag får då ett 200 svar tillbaka.

Däremot om administrator faktiskt existerar så kommer `TO_CHAR` försöka beräkna `1/0`. Eftersom det inte är möjligt att dividera någonting med 0 kommer ett databasfel ske, vilket då ger mig svaret 500.

Detta gör att jag kan ställa ja eller nej frågor och om svaret är:

 - SANT=500
 - FALSKT=200

<img width="1259" height="687" alt="image" src="https://github.com/user-attachments/assets/18c63408-4ff8-4626-b2e5-36e200245fc1" />

<img width="1262" height="673" alt="image" src="https://github.com/user-attachments/assets/724844eb-f2ad-459b-b6ee-4a4c6606ecf6" />

Eftersom jag får 500 (sant) med `administrator` och 200 (falskt) med namnet som garanterat inte existerar. Verifierar det att användaren `administrator` faktiskt existerar.

Nästa steg var då att verifiera hur många tecken som ingår i `administrator` användarens lösenord.

<img width="1263" height="680" alt="image" src="https://github.com/user-attachments/assets/78e0ecf7-4193-4b5d-bbd8-322994785c63" />

Vad detta gör är att fråga databasen om lösenordet är längre än ett tecken.

Jag fick ett 500-svar vilket indikerar att SQL-frågan var sann och lösenordet då är längre än ett tecken.

För att vara säker på att databasen faktiskt kan identifiera lösenordet och längden på det så ställde jag därefter en SQL-fråga med 
ett väldigt högt nummer istället för att svaret skulle bli falskt.

<img width="1307" height="679" alt="image" src="https://github.com/user-attachments/assets/d986e523-7d6b-4a6a-b6d9-5ee9a7a2ca2f" />

Detta gav istället ett 200 svar vilket indikerar att databasen faktiskt kan identifiera information om `administrators` lösenord.

Nästa steg var då att komma fram till exakt hur många tecken lösenordet innehöll. 

<img width="1306" height="679" alt="image" src="https://github.com/user-attachments/assets/cb95c81f-46af-4746-9f9f-82c21fd94251" />

<img width="1308" height="677" alt="image" src="https://github.com/user-attachments/assets/bffeb739-6c77-4e45-9f70-f95df76697c8" />

Eftersom `LENGTH(password)>=20` gav ett 500-svar medan `LENGTH(password)>=21` gav ett 200 svar, så bekräftar det att lösenordet är 20 tecken långt.

Med längden på lösenordet bestämt kunde jag skicka vidare HTTP-requesten till Burp Intruder. 

Som i de flesta portswigger labbar är lösenordet endast små bokstäver a-z och siffror. Därför skapade jag en ordlista med a-z och 0-9

<img width="1642" height="429" alt="image" src="https://github.com/user-attachments/assets/1a196458-7363-4b80-9ca7-a88bf57111f4" />

Vad detta gör är att kolla ifall tecknet på position 1 i lösenord är `a` och `§§` markerar en `payload position` i Burp Intruder. Det innebär att bokstäven `a` i 
detta fall kommer att bytas ut med tecken från min ordlista.

<img width="1049" height="707" alt="image" src="https://github.com/user-attachments/assets/29734524-0fbc-4ca3-aeb3-81d1af302e69" />

Här ser man att alla requests ger ett 200 svar förutom en som är ett 500-svar. Det innebär att `l` är första tecknet i lösenordet.

<img width="1642" height="427" alt="image" src="https://github.com/user-attachments/assets/f4500313-591f-4867-ba57-437251ab1660" />

Denna request kommer då kolla tecknet i position 2 mot min ordlista.

Sedan fortsatte jag att göra detta med alla 20 positioner i lösenordet.

Hade jag haft Burp Pro istället för Community så hade en Cluster Bomb attack varit ett bättre val än Sniper, eftersom jag hade då kunnat sätta en till payload position. I detta fellet på siffran som representerar tecken-position i lösenordet. `SUBSTR(password,§1§,1)`. Jag hade kunnat använda ordlistan 1-20 och därmed automatiserat attacken. 

Eftersom jag använder Burp Community Edition där rate limiting är aktiverat och jag inte ville vänta på en långsammare automatiserad attack valde jag istället att manuellt ändra positionen och köra en Sniper-attack för varje tecken.

Till slut när jag testat samtliga 20 positioner så hade jag ett lösenord: `lzto1nlfgj5r623mb06t`.

Med detta kunde jag logga in på `administrator` kontot.

<img width="1246" height="827" alt="image" src="https://github.com/user-attachments/assets/34c45311-982f-4076-aaf9-baea434b3f2f" />

## Mitigering 

För att mitigera risken till SQL-injektion så bör man använda parametriserade frågor (prepared statements).

I den här labben används värdet från TrackingId-cookien i en SQL-fråga. Om värdet har byggts direkt in i SQL-frågan kan man manipulera SQL-syntaxen.

Med en parametriserad fråga behandlas däremot TrackingId som data och inte som en del av SQL-syntaxen. Det spelar då ingen roll vad man försöker lägga till efter cookien, eftersom innehållet behandlas som ett vanligt värde och inte kan ändra strukturen på SQL-frågan.

