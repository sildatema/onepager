# API Documentatie — Widget Reservation Endpoints

**Bestand:** `api.php`
**Basis-URL:** `/widget/api.php{PATH}`
**Architectuur:** Pooled Reservation — bij boeken (`/submit`) wordt een reservering als `pooled` weggeschreven zonder tafelkoppeling. Pas bij het zetten (`/seat`) wordt een tafel echt gereserveerd (`reservations_tables`) en de status naar `seated` gezet.

---

## Authenticatie (alle endpoints)

Elke request vereist twee query-parameters:

| Param | Type | Omschrijving |
|---|---|---|
| `accessKey` | string | API-sleutel van de klant, wordt gematcht tegen `wpyd_usermeta.user_api_key` |
| `origin` | string | Domein van de aanvrager, wordt gematcht tegen `wpyd_users.user_url` (of whitelisted app-domeinen) |

Ontbreekt of klopt een van beide niet → **403** met `{"error": "..."}`.

CORS: `Access-Control-Allow-Origin` wordt dynamisch gezet op basis van de meegegeven `origin`.

---

## GET /config

Haalt widget-styling en algemene client-configuratie op.

**Query params:** `accessKey`, `origin`

**Response 200**
```json
{
  "button_message": "string",
  "dropdown_color": "#ffffff",
  "backgroundColor": "#ffffff",
  "text_color": "#000000",
  "button_color": "#000000",
  "client_base_url": "https://...",
  "borderRadius": "50px",
  "client_id": 123
}
```

**Errors:** 403 (ongeldige key/origin)

---

## GET /data

Haalt alle data op die de widget nodig heeft om beschikbaarheid te berekenen (tafels, reserveringen, openingstijden, limieten).

**Query params:** `accessKey`, `origin`

**Response 200**
```json
{
  "closed_dates": [],
  "table_groups": [ { "id": 1, "min": 5, "max": 10, "tables": [3,4] } ],
  "buffer": 15,
  "timelimit": 90,
  "all_tables": [ { "id": 1, "min_people": 2, "max_people": 4, "Active": 1 } ],
  "linked_tables": [ /* reservations_tables JOIN Reservations — alleen 'seated' */ ],
  "all_reservations": [ /* pooled + seated + geplaned, zonder tafelkoppeling */ ],
  "excluded_dates": { "ma": "17:00-22:00", "di": "gesloten", ... },
  "labels": [ { "id": 1, "name": "VIP", "abbreviation": "V", "color": "#ff0000" } ],
  "max_days": 60,
  "max_reservations": 0,
  "max_guests": 0,
  "min_time": 0
}
```

**Belangrijk voor de frontend:**
- `linked_tables` → gebruiken als `table_times` (echte, harde conflicten)
- `all_reservations` → gebruiken als `all_reservations` voor shadow assignment (`buildEffectiveTableTimes`)
- `excluded_dates` bevat ondanks de naam de **openingstijden per weekdag** (`state.openingstijden`), niet een lijst uitgesloten datums
- Er wordt **geen `specialDates`/`special_dates` veld** teruggegeven door dit endpoint. Als `reservation-utils.js` `setSpecialDates(...)` verwacht te kunnen aanroepen met data uit `/data`, mist die key hier — dat moet als los meta-veld (bv. `special_dates`) worden toegevoegd aan de query in `widget_get_data()` als je dat wilt gebruiken.

**Errors:** 403, 500

---

## POST /submit

Fase 1 — maakt een nieuwe reservering aan. Standaard als `pooled` (geen tafel gekoppeld). Alleen als de admin expliciet `table_ids` meegeeft, wordt direct als `seated` weggeschreven (quick-seat).

**Query params:** `accessKey`, `origin`

**Body (JSON)**

| Veld | Verplicht | Type | Omschrijving |
|---|---|---|---|
| `selectedPersons` | ja | int (≥1) | Aantal personen |
| `reservation_start_time` | ja | string (parsebaar door `strtotime`) | Starttijd |
| `reservation_end_time` | nee | string | Eindtijd (informatief, wordt herberekend bij quick-seat) |
| `inputDate` | ja | string (datum) | Reserveringsdatum |
| `name` | ja | string | Naam gast |
| `email` | ja | string | Geldig e-mailadres |
| `phone` | ja | string | Telefoonnummer |
| `notes` | nee | string | Opmerkingen |
| `label` | nee | int | Overschrijft automatisch bepaald label |
| `table_ids` | nee | int[] | **Alleen admin quick-seat** — slaat shadow-check over en zet direct op `seated` |

**Validatie / mogelijke 400-fouten**
- `invalid_persons` — `selectedPersons < 1`
- `missing_name`, `missing_email`, `missing_phone`
- `invalid_email`
- `invalid_date_or_time`
- `past_date` — datum ligt in het verleden
- `persons_exceed_max` (incl. `max_guests` in response)
- `time_outside_hours` (incl. `open` in response)
- `invalid_table_ids` — meegegeven `table_ids` horen niet bij deze client of zijn inactief

**409-fouten (slot vol)**
- `slot_full` met `reason`: `max_reservations_reached`, `max_guests_reached`, of `no_tables_available`
- `closed_date` — restaurant gesloten op gekozen datum

**Response 200**
```json
{
  "status": "ok",
  "reservation_id": 456,
  "reservation_status": "pooled",
  "tables": [],
  "label_used": 2
}
```
> Bij quick-seat: `reservation_status` = `"seated"`, `tables` bevat de gekoppelde tafel-ID's.

**Bijwerkingen:**
- Schrijft rij in `Reservations`
- Bij quick-seat: schrijft rij(en) in `reservations_tables`
- Plant e-mails in `Email_bank`: confirmation/Reservation-made, no-show, reminder, evt. review

---

## POST /seat

Fase 2 — zet een `pooled` reservering definitief op een tafel (status → `seated`). Wordt aangeroepen vanuit het dashboard als personeel op "zetten" klikt.

**Query params:** `accessKey`, `origin`

**Body (JSON)**

| Veld | Verplicht | Type | Omschrijving |
|---|---|---|---|
| `reservation_id` | ja | int | ID van de te zetten reservering |
| `table_ids` | nee | int[] | Handmatige tafelkeuze door personeel; anders auto-derive |

**Validatie / fouten**
- `invalid_reservation_id` — 400
- `not_found` — 404 (bestaat niet of hoort niet bij deze client)
- `already_seated` — 409
- `invalid_table_ids` — 400 (handmatige keuze ongeldig)
- `no_tables_available` — 409 (geen enkele tafel past, handmatig toewijzen nodig)

**Response 200**
```json
{
  "success": true,
  "reservation_id": 456,
  "status": "seated",
  "table_ids": [3],
  "start_time": "19:00:00",
  "end_time": "20:30:00"
}
```

**Let op:** het tafelconflict wordt hier gecontroleerd tegen **echte** `reservations_tables`-rijen (niet tegen de pool), omdat een pooled reservering per definitie nog geen tafel bezet houdt.

---

## POST /update

Wijzigt een bestaande reservering (gegevens en/of tafelkoppeling).

**Query params:** `accessKey`, `origin`

**Body (JSON)** — alle velden optioneel, vult aan op bestaande waarden:

| Veld | Type |
|---|---|
| `Reservation_id` | int (verplicht) |
| `Name`, `Email`, `Phone`, `Note` | string |
| `Reservation_date`, `Reservation_time` | string |
| `Reservation_guests` | int |
| `label` | int |
| `team_member` | string → wordt opgeslagen als `employee_name` |
| `table_ids` | int[] — vervangt volledige tafelkoppeling |

**Response**
```json
{
  "success": true,
  "message": "Reservation updated successfully",
  "new_end_time": "20:30:00",
  "tables_added": [4],
  "tables_removed": [3]
}
```
Bij fout: `{"success": false, "message": "..."}`

**Gedrag:**
- Als een `pooled` reservering hier tafels krijgt toegewezen, wordt de status automatisch naar `seated` gepromoveerd.
- Bestaande tafelkoppelingen worden bijgewerkt met nieuwe datum/tijd wanneer die wijzigen.

---

## POST /delete

Verwijdert een reservering volledig (inclusief tafelkoppelingen en geplande e-mails) en stuurt een annuleringsmail.

**Query params:** `accessKey`, `origin`

**Body (JSON)**

| Veld | Verplicht |
|---|---|
| `Reservation_id` | ja |

**Response**
```json
{
  "success": true,
  "message": "Reservation deleted and cancellation email sent",
  "deleted_reservation_id": 456
}
```
Bij fout: `{"success": false, "message": "..."}`

---

## Overzichtstabel

| Endpoint | Methode | Auth | Wijzigt DB | Status-effect |
|---|---|---|---|---|
| `/config` | GET | ✔ | nee | — |
| `/data` | GET | ✔ | nee | — |
| `/submit` | POST | ✔ | ja | maakt `pooled` (of `seated` bij quick-seat) |
| `/seat` | POST | ✔ | ja | `pooled` → `seated` |
| `/update` | POST | ✔ | ja | evt. `pooled` → `seated` |
| `/delete` | POST | ✔ | ja | verwijdert reservering |

---

## Openstaand aandachtspunt (voor overdracht)

`reservation-utils.js` heeft een `setSpecialDates(dates)` setter die `state.specialDates` vult (gebruikt in `getOpeningHoursForDate` om per-datum openingstijden te overschrijven de normale weekdag-tijden). **Het `/data` endpoint levert momenteel geen veld dat hiervoor bedoeld is** — er is alleen `closed_dates` en `excluded_dates` (= weekdag-openingstijden, verwarrende naam). Wie dit overneemt moet beslissen of:
1. `setSpecialDates` nog ergens anders gevoed wordt (bv. los in de proxy), of
2. `widget_get_data()` in `api.php` moet worden uitgebreid met een `special_dates`-meta-veld dat overeenkomt met `Client_id`-specifieke uitzonderingsdagen.
