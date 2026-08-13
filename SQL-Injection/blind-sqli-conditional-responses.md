# Lab: Blind SQL injection with conditional responses

This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and no error messages are displayed. But the application includes a Welcome back message in the page if the query returns any rows.

The database contains a different table called users, with columns called username and password. You need to exploit the blind SQL injection vulnerability to find out the password of the administrator user.

To solve the lab, log in as the administrator user.

## Lösning

När man tar sig till sidan `Home` så får man meddelandet `Welcome back!`

Jag fångade HTTP GET Request till `Home` och skickade till Burp Repeater.

I response-sektionen så sökte jag efter nyckelordet `Welcome` och fick en match.

<img width="1264" height="845" alt="bild" src="https://github.com/user-attachments/assets/25423e65-32c6-40a3-b68e-c996d263b06b" />

SQL-injektionssårbarheten finns i sidans `TrackingId` cookie.

För att verifiera detta började jag med att testa ett förhållande som alltid är sant: `AND '1'='1`.

<img width="1262" height="835" alt="bild" src="https://github.com/user-attachments/assets/54842743-9d4e-4794-9967-b994ed7ab58a" />

Jag verifierade att `Welcome back!` meddelandet fortfarande visades.

Därefter provade jag ett förhållande som aldrig är sant: `AND '1'='2`.

<img width="1266" height="845" alt="bild" src="https://github.com/user-attachments/assets/0fd41f9f-f241-4e92-a532-97474cb969a4" />

Jag verifierade att `Welcome back!` meddelandet inte längre fanns med i response-sektionen.

Detta indikerar att jag kan verifiera information från databasen genom att ställa frågor och få ett svar beroende på om villkoret är sant eller falskt.

Jag verifierar sedan att tabellen `users` existerar.

<img width="1262" height="844" alt="bild" src="https://github.com/user-attachments/assets/6e0c10cf-1b37-47fc-9104-4c30f85055c8" />

Därefter verifierar jag att användaren `administrator` finns med i tabellen `users`

<img width="1266" height="842" alt="bild" src="https://github.com/user-attachments/assets/1c08b00b-68f8-424a-8d8d-f94b0f775c15" />

Nästa steg var att enumerera hur många tecken som ingick i lösenordet för `administrator`

```sql
AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>=1)='a
```

Detta måste göras tills förhållandet inte längre blir sant. `LENGTH(password)>=20` var fortfarande sant medan `LENGTH(password)>=21` var falskt, vilket betyder att lösenordet är 20 tecken långt.

<img width="1262" height="842" alt="bild" src="https://github.com/user-attachments/assets/af2d0031-20c1-4a14-a891-999b5d302089" />


<img width="1265" height="845" alt="bild" src="https://github.com/user-attachments/assets/ef4123de-8885-4bbb-85ec-8bc3744dc2cf" />

Sedan skickade jag samma HTTP GET-request till Burp Intruder. Min payload var 

```sql
AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a
```

Jag satte en `Payload position` på `a` i slutet av min payload. Därefter skapade jag en ordlista med bokstäverna a-z och siffrorna 0-9, slutligen under `Grep - Match` så lade jag till mitt nyckelord `Welcome`.


<img width="1087" height="266" alt="bild" src="https://github.com/user-attachments/assets/49400424-f701-4058-bed0-752e5216ba94" />


<img width="452" height="312" alt="bild" src="https://github.com/user-attachments/assets/93122644-df08-4a59-a32f-790d602329a8" />


<img width="489" height="421" alt="bild" src="https://github.com/user-attachments/assets/ba194cd1-5639-401a-973d-8cd931d2d926" />

Attacken kommer då testa alla tecken i min ordlista mot första positionen i lösenordet vilket i detta fall matchade med bokstaven `f`.


<img width="1433" height="806" alt="bild" src="https://github.com/user-attachments/assets/11e48206-b293-449a-ac9a-3531cd7381da" />

Jag behövde därefter ändra payloaden till 
```sql
AND (SELECT SUBSTRING(password,2,1) FROM users WHERE username='administrator')='a
```
För att testa andra positionen i lösenordet och fortsätta tills jag hade hittat en matchning för alla 20 positioner.

Slutligen så fick jag fram lösenordet `fkjzcapznoveacipvbew` vilket jag kunde använda för att logga in på kontot `administrator`.

<img width="1254" height="801" alt="bild" src="https://github.com/user-attachments/assets/c3bfe494-fa06-4241-bcf5-b946887647df" />


## Förklaring

Labben anger att tabellen users existerar, att användaren administrator finns och att lösenordet endast består av små bokstäver och siffror.

I ett verkligt scenario hade man behövt enumerera mycket längre och troligtvis automatisera enumereringen av tabeller och användare med hjälp av Intruder.

### Identifiering av sårbarheten

Sårbarheten finns i sidans `TrackingId`-cookie. Sidan använder värdet från cookien i en SQL-fråga, men resultatet visas inte direkt. 

Däremot så ges en indikation på resultatet genom meddelandet: 

`Welcome back!`

Ifall SQL-frågan returnerar minst en rad så visas detta meddelande och om frågan inte returnerar någon rad så visas det inte.

Det innebär att jag kan använda SQL-frågan till att ställa enkla `TRUE` eller `FALSE` frågor och observera svaret i HTTP-responsen.

Därför fångade jag HTTP GET-Requesten för `Home` och skickade den till Burp Repeater. I response-sektionen sökte jag efter `Welcome` för att enkelt kunna se om meddelandet fanns i svaret.

Först testade jag ett villkor som alltid är sant:
```
AND '1'='1
```
Notera att det inte krävs en apostrof efter sista `1` eftersom det redan är en apostrof där i den normala SQL-frågan

Eftersom villkoret är sant fortsätter den ursprungliga SQL-frågan att returnera en rad, vilket gör att `Welcome back!` fortfarande finns kvar i HTTP-responsen.

Därefter använde jag ett villkor som alltid är falskt:
```
AND '1'='2
```
Detta gör att `Welcome back!` försvinner från responsen.

På grund av skillnaden mellan dessa två requests bekräftar jag att jag kan påverka SQL-frågan genom `TrackingId` och se resultatet.

Det är detta som gör sårbarheten till en blind SQL injection. Jag får inte tillbaka exempelvis ett lösenord eller resultatet från en SELECT direkt. 
Istället måste jag ställa en rad frågor där svaret endast kan vara sant eller falskt.

### Identifiering av tabell och användarnamn

Nästa steg var att använda samma metod för att hitta information från databasen.

Jag började med att kontrollera att tabellen users faktiskt fanns.

```sql
AND (SELECT 'a' FROM users LIMIT 1)='a
```
När detta returnerar `Welcome back!` blir villkoret sant vilket indikerar att det finns en tabell vid namn `users`. 

SQL-frågan hämtar alltså `a` från tabellen `users` och jämför det med `a`.

Frågan är alltså i praktiken:
```sql
AND 'a'='a'
```
ifall tabellen `users` faktiskt existerar så hämtas ett `a` och villkoret blir sant.

Däremot om tabellen inte hade existerat hade inget `a` kunnat hämtas från tabellen och villkoret hade då blivit falskt.

Jag använde sedan samma metod för att verifiera att användaren `administrator` existerade.
```sql
AND (SELECT 'a'
FROM users
WHERE username='administrator')='a
```
Detta returnerade som sant vilket verifierade att användaren `administrator` existerade.

### Identifiering av lösenordets längd

När jag identifierat ett användarnamn kunde jag börja enumerera lösenordet.

Först behövde jag ta reda på lösenordets längd. Jag använde `LENGTH()`-funktionen och testade olika värden.
```sql
AND (SELECT 'a' FROM users WHERE username='administrator' AND LENGTH(password)>=1)='a
```

SQL-frågan blir då i praktiken `"Är lösenordets längd minst ett tecken?"`

ifall villkoret är sant returneras `Welcome back!` igen.

Jag ökade därefter värdet stegvis:

`LENGTH(password)>=20`

Gav ett sant resultat, medan: 

`LENGTH(password)>=21`

gav ett falskt resultat.

Det innebär att lösenordet är exakt 20 tecken långt.

### Attack mot lösenordet

När längden på lösenordet var identifierat kunde jag börja testa varje position i lösenordet individuellt.

Jag använde SQL-funktionen SUBSTRING():

```sql
AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a
```

Funktionen `SUBSTRING(password,1,1)` innebär här att jag hämtar ett tecken från position 1 i lösenordet.

Frågan blir då i praktiken: `"Är det första tecknet i lösenordet 'a'?`

Ifall svaret är ja så kommer `Welcome back!` att returneras i HTTP-response.

Jag använde Burp Intruder med en `Sniper`-attack för att automatisera processen.

### Intruder konfigurering

Jag markerade karaktären `a` i slutet av queryn som `payload-position`:

```sql
AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='§a§
```
I denna labb fick man veta på förhand att lösenordet skulle endast innehålla små bokstäver och nummer 0-9. 

Därför behövde min ordlista endast innehålla:

a-z

0-9

Under `Grep - Match` lade jag till: `Welcome`

Det gör att Intruder kommer markera resultatet som innehåller `Welcome back!`

För den första positionen testades ordlistan mot databasen. Bokstaven `f` markerades med `Welcome` vilket innebär att det första tecknet i lösenordet är `f`.

Därefter flyttade jag `SUBSTRING()` till nästa position i lösenordet.

`SUBSTRING(password,2,1)` hämtar tecknet från position 2.

Sedan upprepade jag processen för samtliga 20 positioner.

Position 1: SUBSTRING(password,1,1) 
Position 2: SUBSTRING(password,2,1) 
Position 3: SUBSTRING(password,3,1)

Varje position testades med ordlistan. De resultat som returnerade `Welcome` skrev jag ner i ordning.

### Resultat 

Efter att jag testat samtliga 20 positioner fick jag fram följande lösenord: 

`fkjzcapznoveacipvbew`

Jag kunde därefter använda lösenordet för att logga in på användaren `administrator`.

### Alternativ

#### Cluster Bomb

För att automatisera attacken ytterligare så hade man kunnat skapa en `Cluster bomb`-attack i Intruder istället för `Sniper`

Då hade man kunnat sätta en `payload-position` på SUBSTRING(password,§1§,1) där ordlistan är 1-20, Samt behållit `payload-position` på `a`.

Detta hade i praktiken gjort att man slipper ändra värdet på `SUBSTRING()` manuellt.

<img width="1596" height="898" alt="bild" src="https://github.com/user-attachments/assets/aacd4c66-5dcc-40ca-9c06-f770ca1d0ac1" />

Eftersom att jag använder Burp Suite Community Edition blir attacken mycket långsam på grund av `Rate-limiting`. Därför valde jag att använda en sniper attack vilket gick snabbare.

#### Binary Search

Hade ordlistan varit mycket större än de 36 tecknen som används i detta fall så hade man kunnat använda följande payloads för att minska området man måste söka efter rätt tecken:
```sql
AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')>='a
```
Detta frågar då ifall första tecknet är större än eller lika med `a`.

Nästa fråga hade därefter kunnat vara:
```sql
AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')>='p
```
Detta frågar då ifall första tecknet är större än eller lika med `p`.

Man kan då halvera sökområdet stegvis.



## Mitigering

Sårbarheten går att mitigera genom att använda parametriserade SQL-frågor (prepared statements), vilket bör vara huvudförsvaret.

I den här labben används värdet från `TrackingId`-cookien i en SQL-fråga. Om värdet har byggts direkt in i SQL-frågan kan man manipulera SQL-syntaxen.

Med en parametriserad fråga behandlas däremot `TrackingId` som data och inte som en del av SQL-syntaxen. Ett värde som:

`AND '1'='1'` 

Hade då behandlats som data och inte kunnat påverka SQL-frågans logik.

Prepared statements bör vara den primära åtgärden men det hade även varit lämpligt för denna miljö att följa principen om "least privilege" genom att begränsa databasanvändarens privilegier. 

Eftersom användningen av `TrackingId` inte kräver tillgång till tabellen `users`, där användares lösenord lagras, hade åtkomsten till denna tabell kunnat begränsas.

Om användaren av databasen endast hade haft de nödvändiga rättigheterna hade informationen man kan komma åt vid en SQL-injektion begränsats.




























