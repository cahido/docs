# Documentatie

Deze documentatie bevat een kort overzicht van de acties die je kunt uitvoeren met de webservice.

Om te beginnen, zorg dat je het volgende klaar hebt:
- De invoerdata in de vorm van het afgesproken format (Excel bestand).
- De API key om toegang te krijgen tot de webservice op `camas.cahido.nl`.
- Een tool om webrequests te maken, zoals `curl` of Postman. Uiteraard is het ook mogelijk om een
  eigen script te schrijven in bijvoorbeeld Python om het stappenplan te automatiseren.

Een meer gedetailleerde uitleg van de API is te vinden in de [interactieve OpenAPI specificatie](https://camas.cahido.nl/docs).
Je kan hier ook direct requests maken en de API uitproberen door de API token in te vullen bij de `Authorize` knop.

## 0. De data

Let bij het invullen van de data op de volgende belangrijke punten:
- Zorg ervoor dat data (met name jaartallen!) kloppen met het bereik van de planningsperiode. 
  Let hier met name op bij jaarlijks terugkerende evenementen zoals vakanties.
- Zorg ervoor dat sleutels altijd volledig ingevuld zijn. Dit gaat dan bijvoorbeeld om
  (afstanden tussen) adressen, labels voor mogelijke trainers / locaties, personeelsnummers in 
  agendas etc. In het bijzonder gaat dit om de volgende kolommen:
  - `training` in het tabblad `sessies` moet overeenstemmen met kolom A in `deelnemers`.
  - `voorkeursdag` in het tabblad `sessies` mag een of meerdere van `ma`/`di`/`wo`/`do`/`vr` bevatten, gescheiden door een ampersand (`&`).
  - `deelnemersgroep` in het tabblad `sessies` moet overeenstemmen met kolom A en rij 1 in tabblad `overlap`.
  - `mogelijke_locaties` in het tabblad `sessies` moet overeenstemmen met `labels` in tabblad `locaties`. Dit geeft aan
    welke labels een locatie allemaal moet hebben (dus `met_beamer & met_whiteboard` betekent dat alleen locaties 
    met beide labels geschikt zijn voor deze training).
  - `mogelijke_trainers` in het tabblad `trainers` moet overeenstemmen met `labels` in het tabblad `trainers`.
    Wederom geldt hier: alleen trainers met _alle_ genoemde labels zijn geschikt voor deze training.
  - `vestiging` in het tabblad `vestigingen` moet overeenstemmen met rij 1 van `deelnemers`.
  - Alle waardes in kolom `adres` in het tabblad `vestigingen` moeten ook bestaan in kolom A en rij 1 van `reistijd`.
  - `personeelsnummer` in het tabblad `trainers` moet overeenstemmen met kolom `personeelsnummer` in `trainer-vrij`.
  - Alle waardes in kolom `postcode` in tabblad `trainers` moeten ook bestaan in kolom A en rij 1 van `reistijd`.
  - `locatie` in het tabblad `locaties` moet overeenstemmen met kolom `locatie` in tabblad `locatie-vrij`.

## 1. Overzicht

Je kan een overzicht van de taken die in het verleden gestart zijn opvragen met het volgende commando:

```commandline
curl -X GET -L https://camas.cahido.nl/ -H "Authorization: Bearer <API Token>"
```
dus bijvoorbeeld

```commandline
curl -X GET -L https://camas.cahido.nl/ -H "Authorization: Bearer abcdefghijklmnopqrstuvwxyz"
```

Met het antwoord

```json
{
    "tasks": [
        {
            "id": "abcdef12-1234-5678-abcd-abcdef123456",
            "started": "2025-09-24T13:40:10.844033",
            "status": "PENDING"
        },
        ...
    ],
    "msg": "List of tasks"
}
```

waarbij `started` de starttijd is (met tijdzone UTC, dus dit zou een of twee uur kunnen 
verschillen met de lokale tijd) en `status` de huidige status van de taak. De mogelijke statussen 
zijn:
- `PENDING`: De taak staat in de wachtrij en zal binnenkort worden uitgevoerd.
- `RUNNING`: De taak wordt momenteel uitgevoerd.
- `SUCCESS`: De taak is succesvol voltooid.
- `FAILURE`: De taak is mislukt.
- `EXPIRED`: De taak is te oud en uit onze omgeving verwijderd.

## 2. Start een solve

Je start een solve met

```commandline
curl -X POST -L https://camas.cahido.nl/ -H "Authorization: Bearer <API Token>" -F "file=@<Pad naar bestand>"
```

Bijvoorbeeld, met API token `abcdefghijklmnopqrstuvwxyz` en een bestand `C:/pad/naar/dataset.xlsx`, voer je het volgende commando uit:

```commandline
curl -X POST -L https://camas.cahido.nl/ -H "Authorization: Bearer abcdefghijklmnopqrstuvwxyz" -F "file=@C:/pad/naar/dataset.xlsx"
```

Dit commando uploadt het Excel bestand naar de webservice en start het planningsproces. 

### 2a. De solve wordt gestart
Je krijgt hieruit `200 OK` response met een JSON object met de volgende waardes:

```json
{
  "task_id": "abcdef12-1234-5678-abcd-abcdef123456"
}
```

De taak ID is een unieke identifier voor deze specifieke solve. Bewaar deze goed, want je hebt
hem nodig om de status van de taak te controleren en het resultaat op te halen.

### 2b. Er draait al een solve

Als er al een lopende taak is, dan krijg je een `429 Too Many Requests` response met de volgende inhoud:

```json
{
    "detail": "Too many active tasks. Please wait for existing tasks to complete."
}
```

In dat geval zou je uit het overzicht (zie stap 1) de meest recente taak kunnen opzoeken, en 
hiervan de `task_id` gebruiken om de status te volgen en later het resultaat op te halen.

## 3. Volg de status van een taak

Volg de status van een taak met het volgende commando:

```commandline
curl -X GET -L https://camas.cahido.nl/status/<task_id> -H "Authorization: Bearer <API Token>"
```

Waarbij `<task_id>` de taak ID is die je in de vorige stap hebt ontvangen. Bijvoorbeeld:

```commandline
curl -X GET -L https://camas.cahido.nl/status/abcdef12-1234-5678-abcd-abcdef123456 -H "Authorization: Bearer abcdefghijklmnopqrstuvwxyz"
```

Je krijgt een `200 OK` response met een JSON object met de volgende waardes:

```json
{
  "status": "RUNNING",
  "msg": "Task is currently being processed."
}
```

De mogelijke statussen zijn hetzelfde als hierboven beschreven. Blijf dit commando herhalen 
totdat de status `SUCCESS` of `FAILURE` is (`EXPIRED` zal in principe nooit voorkomen, tenzij
je een verkeerde `task_id` gebruikt, maar is weliswaar ook goed om te checken). 

## 4. Haal de logs op

Zodra de status `SUCCESS` of `FAILURE` is, kan je de logs ophalen van de taak.
Dit doe je door het volgende commando uit te voeren.

```commandline
curl -X GET -L https://camas.cahido.nl/logs/<task_id> -H "Authorization: Bearer <API Token>"
```

dus bijvoorbeeld

```commandline
curl -X GET -L https://camas.cahido.nl/logs/abcdef12-1234-5678-abcd-abcdef123456 -H "Authorization: Bearer abcdefghijklmnopqrstuvwxyz"
```

en je krijgt een antwoord

```json
{
    "messages": [
        "[WARNING] 2025-09-24 12:51:45,601 - Geen beschikbaarheid voor een van de locaties",
        ...
    ]
}
```

## 5. Haal het resultaat op
Zodra de status `SUCCESS` is kan je van een taak het resultaat ophalen (ook in het 
overeengekomen format). Als de status `SUCCESS` is, kan je het resultaat ophalen met

```commandline
curl -f -X GET -L https://camas.cahido.nl/result/<task_id> \
    -H "Authorization: Bearer <API Token>" \ 
    -o resultaat.zip
```

of met een ander pad als bestandsnaam. Dus bijvoorbeeld

```commandline
curl -f -X GET -L https://camas.cahido.nl/result/abcdef12-1234-5678-abcd-abcdef123456 \
    -H "Authorization: Bearer abcdefghijklmnopqrstuvwxyz" \
    -o resultaat.zip
```

Merk de `-f` optie op, die zorgt dat `curl` een niet succesvolle response (zoals `404 Not Found`)
als een fout behandelt, en in dat geval dus _geen_ bestand download. Je krijgt een `200 OK` 
response die opgeslagen wordt in het opgegeven pad. 

## 6. Een taak annuleren

Je kunt een (lopende) taak annuleren. Hiermee zal een taak gestopt worden, maar _niet_ uit het 
overzicht verwijderd worden. Deze krijgt dan de status `FAILURE`. Voer hiervoor het volgende 
commando uit:

```commandline
curl -X POST -L https://camas.cahido.nl/cancel/<task_id> -H "Authorization: Bearer <API Token>"
```

dus bijvoorbeeld

```commandline
curl -X POST -L https://camas.cahido.nl/cancel/abcdef12-1234-5678-abcd-abcdef123456 -H "Authorization: Bearer abcdefghijklmnopqrstuvwxyz"
```
Je krijgt een antwoord van de vorm
```json
{
  "task_id": "abcdef12-1234-5678-abcd-abcdef123456",
  "msg": "Task cancelled"
}
```

## 7. Een taak verwijderen

Je kunt een taak verwijderen uit het overzicht. Dit zal de taak eerst annuleren als deze nog 
loopt, en daarna verwijderen uit het overzicht. Voer hiervoor het volgende commando uit:
```commandline
curl -X DELETE -L https://camas.cahido.nl/delete/<task_id> -H "Authorization: Bearer <API Token>"
```
dus bijvoorbeeld
```commandline
curl -X DELETE -L https://camas.cahido.nl/delete/abcdef12-1234-5678-abcd-abcdef123456 -H "Authorization abcdefghijklmnopqrstuvwxyz"
```
Je krijgt een antwoord van de vorm
```json
{
  "task_id": "abcdef12-1234-5678-abcd-abcdef123456",
  "msg": "Task deleted"
}
```