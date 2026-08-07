# Lab: SQL injection attack, listing the database contents on non-Oracle databases

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The application has a login function, and the database contains a table that holds usernames and passwords. You need to determine the name of this table and the columns it contains, then retrieve the contents of the table to obtain the username and password of all users.

To solve the lab, log in as the administrator user. 

## Lösning 

Jag använder Burp Suite för att kunna fånga HTTP GET Request av en kategori i Burp Proxy. Därefter skickade jag vidare den till Burp Repeater. På så sätt kan jag prova olika payloads enkelt utan att behöva fånga nya requests.

Jag började med att enumerera antalet kolumner som SELECT-satsen hämtar från databasen.

<img width="1260" height="794" alt="image" src="https://github.com/user-attachments/assets/35c9aaf0-2472-432a-8751-b407ae8ad4f5" />

Eftersom payloaden med två kolumner returnerade ett normalt HTTP-svar medan andra antal kolumner gav fel, indikerar detta att originalfrågan sannolikt innehåller två kolumner.

Därefter enumererade jag vilka tabeller som finns i databasen.

<img width="1261" height="794" alt="image" src="https://github.com/user-attachments/assets/d37595cc-a20d-4c10-acd5-956944a03dfd" />

Jag fick då en lista över alla tabellnamn som finns i databasen, bland dessa hittade jag en intressant tabell vid namn `users_aywgaj`.

<img width="1259" height="792" alt="image" src="https://github.com/user-attachments/assets/036bd277-c40d-4172-a41e-a2af86956835" />

Nästa steg var att enumerera vilka kolumner som finns i `users_aywgaj`.

<img width="1261" height="802" alt="image" src="https://github.com/user-attachments/assets/22bfcd5e-c55f-4742-a422-92831c5253b9" />

I listan av kolumner hittade jag två intressanta kolumner vid namn `password_igpjhi` och `username_ybjzkb`.

Därefter hämtade jag informationen från kolumnerna. Eftersom den ursprungliga SELECT-satsen redan hämtar två kolumner behövs ingen `NULL` som platshållare. Jag kan därför hämta de två kolumner jag behöver direkt.

<img width="1260" height="797" alt="image" src="https://github.com/user-attachments/assets/b712e949-782f-469f-ba02-394973caaefd" />

För användaren `administrator` visades lösenordet `7f7grhkdwwohmvumr16u`.

Jag loggade in med de uppgifter jag hade hämtat.

<img width="1275" height="664" alt="image" src="https://github.com/user-attachments/assets/b716a862-fd6d-4d02-9c8e-d74586bb4861" />

## Förklaring 

I denna labb så får man veta att det inte är en Oracle-databas. 
I ett verkligt scenario så hade man behövt enumerera det först. För en mer detaljerad förklaring se min andra writeup: 

[SQL injection attack, querying the database type and version on Oracle](../SQL-Injection/querying-database-type-and-version.md)




























information_schema är ett systemschema som innehåller data om databasen. 
Till skillnad från vanliga tabeller lagrar det inte applikationens data, utan information om databasens struktur, exempelvis vilka tabeller och kolumner som finns. 
Vid SQL-injektion kan detta användas för att enumerera databasen innan känslig data hämtas.
