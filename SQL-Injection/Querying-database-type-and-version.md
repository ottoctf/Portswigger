# Lab: SQL injection attack, querying the database type and version on Oracle

This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

To solve the lab, display the database version string.

## Lösning

Jag använder Burp Suite för att kunna fånga HTTP GET Request av en kategori i Burp Proxy. Därefter skickade jag vidare den till Burp Repeater för att kunna prova olika payloads enkelt utan att behöva fånga nya requests.

Jag började med att enumerera antalet kolumner som SELECT-satsen hämtar från databasen. 

<img width="1260" height="788" alt="bild" src="https://github.com/user-attachments/assets/033606b5-ad35-49b9-ab72-861651f165e8" />

Eftersom payloaden med två kolumner returnerade ett normalt HTTP-svar medan andra antal kolumner gav fel, indikerar detta att originalfrågan sannolikt innehåller två kolumner.

Därefter använder jag min payload för att hämta databasversionen.

<img width="1259" height="731" alt="bild" src="https://github.com/user-attachments/assets/ed89a59e-d6ab-42da-997b-3c0c9d2b2d30" />

<img width="583" height="129" alt="bild" src="https://github.com/user-attachments/assets/f18d0132-0354-4b8a-aa55-569f257ad5ab" />

## Förklaring

### Identifiering av databastyp och enumerering av kolumner

I denna labb får man veta i förväg att det är en Oracle-databas, men i ett verkligt scenario är detta oftast okänt.

Om databasen exempelvis hade varit MySQL eller PostgreSQL hade följande SQL-fråga kunnat användas för att enumerera antalet kolumner:

```sql
UNION SELECT NULL,NULL
```
I en Oracle-databas ger detta däremot ett 500 Internal Server Error, eftersom Oracle kräver att en SELECT-sats innehåller en FROM-del.

Därför finns det en så kallad "dummy-tabell" som används i Oracle och kallas `dual`. Om man exempelvis behöver köra en SELECT-fråga utan att hämta data från en faktisk tabell kan dual användas som en platshållare.

```sql
UNION SELECT NULL,NULL FROM dual
```
När jag då får en 200 OK efter att jag lade till `FROM dual` så är det en stark indikation på att databasen är Oracle.

### Payload

I en Oracle-databas så innehåller `banner` kolumnen i systemvyn `v$version` versionsinformation för databasen, vilket kan användas för att identifiera databasversionen.

UNION SELECT-satsen måste returnera samma antal kolumner som den ursprungliga SELECT-satsen, vilket i detta fall är två kolumner.

Därför används `NULL` för att endast uppfylla det kravet.

Min SQL-injektions payload blir därför:
```sql
UNION SELECT banner,NULL FROM v$version
```

## Mitigering

Sårbarheten kan förhindras genom att använda parameteriserade SQL-frågor (prepared statements). Då behandlas användarens inmatning som data i stället för som om det vore en del av SQL-frågan. Det förhindrar att en angripare 
kan ändra frågans struktur med exempelvis en apostrof.
