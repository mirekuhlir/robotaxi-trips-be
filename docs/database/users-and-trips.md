# Users & trips

[← Přehled schématu](README.md)

## Tabulky

### `users`

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `email` | VARCHAR, UNIQUE, NOT NULL | |
| `first_name` | VARCHAR | |
| `last_name` | VARCHAR | |
| `display_name` | VARCHAR | |
| `avatar_url` | TEXT | |
| `timezone` | VARCHAR, NOT NULL | Výchozí `UTC` |
| `locale` | VARCHAR, NOT NULL | Výchozí `en` |
| `home_currency` | CHAR(3), NOT NULL | Výchozí plánovací měna uživatele (ISO 4217 uppercase); při vytvoření výletu se zkopíruje do `trips.home_currency`; výchozí `USD` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

### `trips`

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `created_by` | UUID, FK → users | ON DELETE RESTRICT. **Neměnný** auditní záznam původního tvůrce — žádný UPDATE (aplikace zakazuje, DB nemá trigger ani CHECK). Slouží výhradně pro audit a dotazy na autora; **nepřidává** automaticky oprávnění — přístup vždy jen přes `trip_members`. Tvůrce je při vytvoření vždy `admin` v téže transakci jako INSERT do `trips`. Převod `created_by` na jiného uživatele neexistuje. |
| `title` | VARCHAR, NOT NULL | Název výletu v jazyce autora (autorský obsah, ne systémový překlad) |
| `description` | TEXT, nullable | Autorský popis výletu v jazyce autora (autorský obsah, ne systémový překlad) |
| `cover_image_url` | TEXT, nullable | URL náhledového obrázku pro katalog a detail |
| `status` | trip_status | Výchozí `draft`; `published` = hotový výlet viditelný dle `visibility` |
| `visibility` | trip_visibility | Výchozí `private`; bezpečnostní hranice čtení. `private` = jen členové, `unlisted` = čitelné přes přímý UUID odkaz, `public` = katalogově viditelné při `status = published` |
| `home_currency` | CHAR(3), NOT NULL | Měna, ve které autor chce výlet plánovat a zobrazovat rozpočet (ISO 4217 uppercase); editovatelná na detailu výletu; při vytvoření výletu výchozí z `users.home_currency`; výchozí `USD` |
| `total_cost_usd` | NUMERIC(10, 2) | Agregovaná cena v USD; cache odvozená ze segmentů (`SUM price_usd`); kanon pro veřejný katalog a filtrování napříč výlety; ne ručně editovatelná; výchozí `0` |
| `total_cost_home` | NUMERIC(10, 2) | Agregovaná cena v `home_currency`; cache odvozená ze segmentů (`SUM home_price_amount`); pro rozpočet a zobrazení výletu autorovi; ne ručně editovatelná; výchozí `0` |
| `total_duration_minutes` | INTEGER | Předpočítaná délka aktivit a dopravy v minutách (bez ubytování); cache odvozená ze segmentů, ne ručně editovatelná; výchozí `0`. **Není** doba dojezdu z výchozího místa — viz `destination_place_id` / filtr dojezdu |
| `destination_place_id` | UUID, FK → places, nullable | Cache destinace výletu (`places.id`); odvozená ze segmentů, ne ručně editovatelná; `NULL` = destinaci nelze určit — výlet neprojde filtrem dojezdu. ON DELETE SET NULL |
| `outbound_transport_mode` | transport_mode, nullable | Cache režimu dopravy „jak se tam dostanu“ z itineráře; párový s `destination_place_id` (obě NOT NULL nebo obě NULL). Pravidla odvození — viz [Cache destinace a outbound režimu](travel-times.md#cache-destinace-a-outbound-režimu-na-trips) |
| `recommended_age_min` | SMALLINT, nullable | Nejvyšší minimální věk mezi aktivitami — `MAX(segments.recommended_age_min)` přes `segment_kind = activity` (`NULL` = bez spodní hranice); cache odvozená ze segmentů, ne ručně editovatelná |
| `recommended_age_max` | SMALLINT, nullable | Nejpřísnější horní hranice věku mezi aktivitami — `MIN(segments.recommended_age_max)` přes `segment_kind = activity` (`NULL` = bez horní hranice); cache odvozená ze segmentů, ne ručně editovatelná |
| `max_difficulty` | segment_difficulty, nullable | Agregovaná nejvyšší náročnost z aktivit; cache odvozená ze segmentů, ne ručně editovatelná; `NULL` = žádná aktivita s náročností — ve filtrovaném katalogu dle náročnosti se nezobrazí |
| `temp_min_c` | NUMERIC(4, 1), nullable | Agregovaná nejnižší očekávaná **vzduchová** teplota výletu ve °C; cache z `weather_records` s fallbackem na `weather_climate_months`, ne ručně editovatelná |
| `temp_max_c` | NUMERIC(4, 1), nullable | Agregovaná nejvyšší očekávaná **vzduchová** teplota výletu ve °C; cache z `weather_records` s fallbackem na `weather_climate_months`, ne ručně editovatelná |
| `feels_like_min_c` | NUMERIC(4, 1), nullable | Agregovaná nejnižší očekávaná **pocitová** teplota výletu ve °C; cache z `weather_records` s fallbackem na klima (viz [Pocitová teplota](weather-and-climate.md#pocitová-teplota)); ne ručně editovatelná; primární pro katalogový filtr teploty |
| `feels_like_max_c` | NUMERIC(4, 1), nullable | Agregovaná nejvyšší očekávaná **pocitová** teplota výletu ve °C; cache z `weather_records` s fallbackem na klima; ne ručně editovatelná; primární pro katalogový filtr teploty |
| `temperature_source` | temperature_source, nullable | Odkud teplotní cache pochází: `weather_records` (všechny dotčené týdny měly týdenní záznam, včetně rodičovské oblasti přes [geo-fallback](weather-and-climate.md#hierarchie-oblastí-a-geo-fallback)) nebo `climate` (alespoň jeden se dopočítal z `weather_climate_months`); `NULL` = teplotní cache je prázdná — viz [Fallback na klima](weather-and-climate.md#fallback-na-klima) |
| `rating_avg` | NUMERIC(2, 1), nullable | Průměrné uživatelské hodnocení výletu (`1.0`–`5.0`); cache z `trip_reviews.score`; ne ručně editovatelná; `NULL` = žádné recenze (`rating_count = 0`) |
| `rating_count` | INTEGER, NOT NULL | Počet uživatelských recenzí; cache z `trip_reviews`; ne ručně editovatelná; výchozí `0` |
| `packing_computed_at` | TIMESTAMPTZ, nullable | Čas posledního přepočtu cache balení (`trip_packing_items`, `segment_packing_items`); `NULL` = cache ještě nebyla spočítána |
| `travel_requirements_computed_at` | TIMESTAMPTZ, nullable | Čas posledního přepočtu cache cestovních požadavků (`trip_travel_requirements`); `NULL` = cache ještě nebyla spočítána |
| `robotaxi_advisories_computed_at` | TIMESTAMPTZ, nullable | Čas posledního přepočtu cache robotaxi upozornění (`trip_robotaxi_advisories`, `segment_robotaxi_advisories`); `NULL` = cache ještě nebyla spočítána |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

### `trip_members`

| Sloupec | Typ | Popis |
|---|---|---|
| `trip_id` | UUID, PK, FK → trips | ON DELETE CASCADE |
| `user_id` | UUID, PK, FK → users | ON DELETE CASCADE |
| `role` | member_role | Výchozí `viewer` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

Při vytvoření výletu aplikace automaticky vloží tvůrce (`trips.created_by`) do `trip_members` s rolí `admin` a nastaví `trips.home_currency` z `users.home_currency` tvůrce (v téže transakci jako INSERT do `trips`). Existující admin může jmenovat další adminy; na jednom výletu může být více adminů.

Admin **může** odstranit původního tvůrce z `trip_members`, pokud na výletu zůstane **alespoň jeden jiný admin**. `trips.created_by` zůstává neměnný auditní záznam bez vlivu na přístup. Invariant: na každém výletu musí být v `trip_members` **alespoň jeden admin** — aplikace validuje při DELETE člena nebo UPDATE role.

Dotaz „Moje výlety“: `SELECT trip_id FROM trip_members WHERE user_id = :current_user` — bez UNION s `created_by` (tvůrce je vždy v `trip_members` při vytvoření; po odstranění z `trip_members` už výlet nevidí).

### `trip_reviews`

Uživatelská recenze výletu — skóre, text a volitelná galerie médií. Jeden aktivní hlas na uživatele a výlet. Agregace do `trips.rating_avg` / `trips.rating_count` (viz [Recenze](#recenze)).

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `trip_id` | UUID, FK → trips | ON DELETE CASCADE |
| `user_id` | UUID, FK → users | ON DELETE RESTRICT; autor recenze |
| `score` | SMALLINT, NOT NULL | Celé hodnocení `1`–`5` |
| `body` | TEXT, nullable | Text recenze v jazyce autora (UGC, ne systémový překlad); prázdný řetězec normalizuj na `NULL` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**UNIQUE** `(trip_id, user_id)` — upsert při změně skóre / textu.

### `trip_review_media`

Galerie médií k recenzi výletu — 0..N řádků na recenzi. Stejný URL pattern jako `segment_images` (upload/CDN/R2 mimo DB).

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `review_id` | UUID, FK → trip_reviews | ON DELETE CASCADE |
| `media_kind` | review_media_kind, NOT NULL | `image` nebo `video` |
| `media_url` | TEXT, NOT NULL | Veřejná HTTPS URL (CDN/R2/S3); stejný pattern jako `trips.cover_image_url` / `segment_images.image_url` |
| `poster_url` | TEXT, nullable | Náhledový obrázek u videa (HTTPS); u `media_kind = image` vždy `NULL` |
| `caption` | TEXT, nullable | Volitelný popisek v jazyce autora; prázdný řetězec normalizuj na `NULL` |
| `sort_order` | SMALLINT, NOT NULL | Pořadí v galerii; výchozí `0`; `>= 0`; řazení `ORDER BY sort_order, id` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

Validace v aplikaci: `media_url` musí být neprázdná HTTPS URL; max. počet médií per recenze volitelně v aplikaci (např. 10), ne v DB — viz [Recenze](#recenze).


---

## Poznámky k implementaci

### Status, visibility a mazání výletů

| Mechanismus | Účel |
|---|---|
| `status` | `draft` = rozpracovaný; `published` = hotový výlet |
| `visibility` | `private` = čitelný jen pro členy; `unlisted` = mimo veřejný katalog, ale čitelný přes přímý UUID odkaz; `public` = může být v katalogu (pokud `published`) |

#### Matice `status` × `visibility`

| status | visibility | Veřejný katalog | Přímý odkaz (UUID) — nečlen |
|---|---|---|---|
| `draft` | `private` | ne | ne |
| `draft` | `unlisted` | ne | ano — jen čtení |
| `draft` | `public` | ne | ano — jen čtení |
| `published` | `private` | ne | ne |
| `published` | `unlisted` | ne | ano — jen čtení |
| `published` | `public` | ano | ano — jen čtení |

Veřejný katalog: `status = 'published' AND visibility = 'public'`. Do filtrovaného katalogu (s filtrem na teplotu nebo náročnost) patří jen výlety s vyplněnou cache `feels_like_min_c` / `feels_like_max_c` resp. `max_difficulty`; výlet bez těchto dat může zůstat dostupný členům nebo přes přímý odkaz podle `visibility`, ale neprojde teplotním ani náročnostním filtrem. Teplotní cache se díky [fallbacku na klima](weather-and-climate.md#fallback-na-klima) vyplní i u výletů mimo horizont prognózy, pokud mají jejich oblasti klimatické normály; `temperature_source` pak nese hodnotu `climate`. Filtr **dojezdu** vyžaduje vyplněnou cache `destination_place_id` + `outbound_transport_mode` a existující odhad v `travel_time_estimates` (nebo stejné místo jako origin) — viz [Katalogový filtr dojezdu](travel-times.md#katalogový-filtr-dojezdu). Katalog podporuje filtrování a řazení dle ceny v **USD** (`total_cost_usd` — kanon pro srovnání napříč výlety; `total_cost_home` se v katalogu nepoužívá), délky programu (`total_duration_minutes`), dojezdu z vybraného místa (`destination_place_id` + `travel_time_estimates`), věku (`recommended_age_min` / `recommended_age_max`), náročnosti (`max_difficulty`), teploty — primárně pocitové (`feels_like_min_c` / `feels_like_max_c`), volitelně vzduchové (`temp_min_c` / `temp_max_c`) — a uživatelského hodnocení (`rating_avg` / `rating_count`; výlety bez recenzí mají `rating_avg = NULL`) — viz [Cena a délka výletu](segments.md#cena-a-délka-výletu), [Travel times](travel-times.md), [Teplota výletu](weather-and-climate.md#teplota-výletu), [Doporučený věk a náročnost](segments.md#doporučený-věk-a-náročnost) a [Recenze](#recenze).

**Přístup přes odkaz:** URL s `trips.id` (UUID v4) stačí bez přihlášení jen pro `visibility IN ('unlisted', 'public')`. UUID slouží jako neuhodnutelný token pro odkazové sdílení; rate-limit volitelně v aplikaci (mimo DB schéma). U `visibility = 'private'` přímý UUID odkaz bez členství nestačí.

`status` není bezpečnostní hranice — rozlišuje rozpracovaný vs. publikovaný obsah. Bezpečnostní hranici čtení určuje `visibility`: nově vytvořený výlet má výchozí `private`, takže draft není omylem veřejně čitelný přes UUID.

- **Nečlen** (není v `trip_members`): u `private` žádný přístup; u `unlisted` a `public` pouze **čtení** itineráře a metadat přes UUID odkaz; žádná editace, žádné DELETE.
- **Člen** dle role v `trip_members` — viz tabulka níže.

| Role | Čtení | Editace segmentů / metadat | Správa členů | Smazání výletu |
|---|---|---|---|---|
| `viewer` | ano | ne | ne | ne |
| `editor` | ano | ano | ne | ne |
| `admin` | ano | ano | ano | ano |

Výlet `published + unlisted` není v katalogu, ale je dostupný komukoli s odkazem (read-only). Draft výlet zůstává soukromý, dokud má `visibility = private`; autor ho může záměrně sdílet přepnutím na `unlisted`.

#### Mazání výletu

Definitivní odstranění = **hard delete**:

```sql
DELETE FROM trips WHERE id = :trip_id;
```

CASCADE smaže `trip_members`, `trip_reviews` (a přes ně `trip_review_media`), `trip_packing_items` (a přes ně `trip_packing_item_sources`), `trip_travel_requirements` (a přes ně `trip_travel_requirement_sources`), `segment_robotaxi_advisories`, `trip_robotaxi_advisories`, `segments`, `segment_images`, `segment_packing_items` (a přes ně `segment_packing_item_sources`) a `transit_details`. Oprávnění: jen uživatel s rolí `admin` v `trip_members` pro daný výlet. Skrytí z katalogu bez omezení odkazového sdílení řeší `visibility = unlisted`; úplné omezení čtení na členy řeší `visibility = private`.

Hard delete místa (`DELETE FROM places`) CASCADE smaže `place_reviews` a přes ně `place_review_media`. Místo s FK z `segments` / `transit_details` blokuje RESTRICT na těchto vazbách — nejdřív odstraň nebo přepoj segmenty. Self-FK `places.robotaxi_access_place_id` je `ON DELETE SET NULL`, ale CHECK last-mile u `via_access_point` smazání access place stejně zablokuje, dokud závislá místa last-mile nepřepojíš nebo nevynuluješ.

#### Mazání uživatelů

`trips.created_by` má **ON DELETE RESTRICT** — uživatele, který je uveden jako tvůrce (`created_by`) alespoň jednoho výletu, nelze fyzicky smazat, dokud **všechny tyto výlety nejsou smazány**. Sloupec `created_by` je neměnný; převod na jiného uživatele neexistuje.

`trip_reviews.user_id` a `place_reviews.user_id` mají **ON DELETE RESTRICT** — uživatele s existujícími recenzemi nelze smazat, dokud se jeho recenze nesmažou (hard delete recenzí včetně médií; po DELETE přepočítat cache hodnocení na cíli).

`trip_members.user_id` zůstává **ON DELETE CASCADE** — při smazání uživatele (který není blokován RESTRICT na `created_by` ani na recenzích) se odstraní jeho řádky v `trip_members` u výletů, kde byl jen členem. Před smazáním uživatele, který je tvůrcem výletů, je nutné tyto výlety smazat (hard delete) — jinak RESTRICT operaci zablokuje.

**Aplikační validace před `DELETE FROM users`:** Projít všechny výlety, kde je uživatel **jediný admin** v `trip_members`. Pokud takový výlet existuje → chyba validace, uživatel se nesmaže. RESTRICT na `created_by` tento případ neřeší — původní tvůrce může být už odstraněn z `trip_members`, ale stále jediný admin. Před smazáním také smazat (nebo jinak vyřešit) řádky v `trip_reviews` / `place_reviews` daného uživatele.

```mermaid
flowchart TD
  deleteUser["DELETE FROM users"]
  checkCreatedBy{"Je created_by u nějakého výletu?"}
  checkLastAdmin{"Je jediný admin na výletu?"}
  restrictBlock["RESTRICT — smaž nejdřív výlety"]
  appBlock["Aplikace — chyba validace"]
  cascadeOk["CASCADE trip_members + DELETE users"]

  deleteUser --> checkCreatedBy
  checkCreatedBy -->|ano| restrictBlock
  checkCreatedBy -->|ne| checkLastAdmin
  checkLastAdmin -->|ano| appBlock
  checkLastAdmin -->|ne| cascadeOk
```

Před `cascadeOk` musí aplikace také smazat (nebo jinak vyřešit) `trip_reviews` / `place_reviews` daného uživatele — jinak RESTRICT na `user_id` blokuje `DELETE FROM users`.

### Recenze

Uživatelské recenze jsou UGC obsah připojený k výletu nebo místu: skóre, text a volitelná galerie obrázků/videí. Nejsou polymorfní — paralelní tabulky `trip_reviews` / `place_reviews` (+ `*_review_media`) se stejným tvarem.

#### Co hodnotí co

| Entita | Zdroj pravdy | Cache agregace | Externí / import |
|---|---|---|---|
| Výlet | `trip_reviews` | `trips.rating_avg`, `trips.rating_count` | — |
| Místo | `place_reviews` | `places.review_rating_avg`, `places.review_rating_count` | `places.rating` (Google Maps–style) — **nesmí** se přepisovat z UGC |

#### Pravidla zápisu (aplikace)

- Recenzi může vytvořit/upravit jen přihlášený uživatel; **UNIQUE** `(trip_id|place_id, user_id)` — změna = upsert stejného řádku.
- **Výlet:** cíl musí být čitelný pro autora recenze a `trips.status = published`. Draft se nehodnotí. `trips.created_by` **nesmí** hodnotit vlastní výlet.
- **Místo:** libovolné existující místo; skóre 1–5.
- `body` a `caption`: prázdný řetězec → `NULL`; text zůstává v jazyce autora (bez systémového překladu v DB).
- `media_url` (a vyplněné `poster_url`) musí být neprázdná HTTPS URL — upload/CDN/R2 mimo DB (stejný pattern jako `segment_images`).
- U `media_kind = image`: `poster_url` musí být `NULL`. U `media_kind = video`: `poster_url` volitelný HTTPS náhled.
- Max. počet médií per recenze volitelně v aplikaci (např. 10), ne v DB.
- Galerie: `ORDER BY sort_order, id`.

#### Přepočet cache

Po INSERT / UPDATE (`score`) / DELETE recenze v téže transakci:

```sql
-- výlet
UPDATE trips SET
  rating_count = (SELECT COUNT(*)::int FROM trip_reviews WHERE trip_id = :trip_id),
  rating_avg = (SELECT ROUND(AVG(score)::numeric, 1) FROM trip_reviews WHERE trip_id = :trip_id)
WHERE id = :trip_id;
-- při count = 0 nastav rating_avg = NULL (AVG přes prázdnou množinu → NULL)

-- místo
UPDATE places SET
  review_rating_count = (SELECT COUNT(*)::int FROM place_reviews WHERE place_id = :place_id),
  review_rating_avg = (SELECT ROUND(AVG(score)::numeric, 1) FROM place_reviews WHERE place_id = :place_id)
WHERE id = :place_id;
```

Žádné recenze → `*_count = 0`, `*_avg = NULL` (DB CHECK to vynucuje).

#### FE chování

- Katalog / karta výletu: `rating_avg` + `rating_count` (např. `4.2 · 18`); bez hlasů „Zatím bez hodnocení“.
- Detail výletu / místa: průměr nahoře, formulář ohodnocení (skóre + text + upload médií), seznam recenzí s galerií.
- U míst zobrazuj UGC (`review_rating_*`) odděleně od externího `places.rating`, pokud je obojí vyplněné.

