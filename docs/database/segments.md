# Segments

[← Přehled schématu](README.md)

## Tabulky

### `segments`

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `trip_id` | UUID, FK → trips | ON DELETE CASCADE |
| `segment_kind` | segment_kind, NOT NULL | Povinný; aplikace nastaví při INSERT/UPDATE |
| `start_time` | TIMESTAMPTZ, NOT NULL | |
| `end_time` | TIMESTAMPTZ, NOT NULL | |
| `start_place_id` | UUID, FK → places | ON DELETE RESTRICT |
| `end_place_id` | UUID, FK → places, nullable | ON DELETE RESTRICT |
| `local_price_amount` | NUMERIC(10, 2), NOT NULL | Místní cena na destinaci; u `segment_kind = accommodation` cena přiřazená konkrétnímu ubytovacímu úseku. Pokud jedna rezervace pokrývá více nočních/check-in/check-out úseků, aplikace cenu uloží buď jednou na první úsek a ostatní nechá `0`, nebo ji deterministicky rozdělí — nikdy ji nezapočítá vícekrát; u bezplatných segmentů `0`; výchozí `0` |
| `local_price_currency` | CHAR(3), NOT NULL | ISO 4217 uppercase kód místní měny; výchozí `USD`; u `local_price_amount = 0` uložit `USD` (auditní konvence — viz [Kurz měny](#kurz-měny)) |
| `exchange_rate_local_to_usd` | NUMERIC(12, 6), NOT NULL | Kurz `local_price_currency → USD` použitý při výpočtu `price_usd`; vždy `> 0`; výchozí `1.0` (i u `local_price_amount = 0`) |
| `price_usd` | NUMERIC(10, 2), NOT NULL | `ROUND(local_price_amount × exchange_rate_local_to_usd, 2)`; přepočítá se při INSERT/UPDATE segmentu; vstup pro agregaci `trips.total_cost_usd` (katalog); výchozí `0` |
| `home_price_amount` | NUMERIC(10, 2), NOT NULL | `ROUND(local_price_amount × exchange_rate_local_to_home, 2)` v měně `trips.home_currency`; přepočítá se při INSERT/UPDATE segmentu nebo změně `trips.home_currency`; vstup pro agregaci `trips.total_cost_home`; výchozí `0` |
| `exchange_rate_local_to_home` | NUMERIC(12, 6), NOT NULL | Kurz `local_price_currency → trips.home_currency` v době uložení; vždy `> 0`; výchozí `1.0` (i u `local_price_amount = 0`) |
| `recommended_age_min` | SMALLINT, nullable | Doporučený minimální věk v letech (`NULL` = bez spodní hranice) |
| `recommended_age_max` | SMALLINT, nullable | Doporučený maximální věk v letech (`NULL` = bez horní hranice) |
| `difficulty` | segment_difficulty, nullable | U aktivit a dopravy explicitní (volitelné); u ubytování vždy `NULL`; vstup pro agregaci `trips.max_difficulty` (jen aktivity) a pro `clothing_rules` na úrovni segmentu |
| `title` | VARCHAR, nullable | Autorský nadpis segmentu v jazyce autora výletu (autorský obsah, ne systémový překlad); `NULL` = FE zobrazí `places.name` u `start_place_id` (pokud je i to `NULL`, generický placeholder) |
| `description` | TEXT, nullable | Autorský popis segmentu v jazyce autora výletu (autorský obsah, ne systémový překlad) |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

Platí pro všechny `segment_kind` (`accommodation`, `activity`, `transit`). Prázdný string normalizuj v aplikaci na `NULL`.

### `segment_images`

Galerie obrázků segmentu — 0..N řádků na segment.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `segment_id` | UUID, FK → segments | ON DELETE CASCADE |
| `image_url` | TEXT, NOT NULL | Veřejná URL (CDN/R2/S3); stejný pattern jako `trips.cover_image_url` |
| `caption` | TEXT, nullable | Volitelný popisek k danému obrázku v jazyce autora |
| `sort_order` | SMALLINT, NOT NULL | Pořadí v galerii segmentu; výchozí `0`; `>= 0`; řazení `ORDER BY sort_order, id` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

Validace v aplikaci: `image_url` musí být neprázdná HTTPS URL; max. počet obrázků per segment volitelně v aplikaci (např. 20), ne v DB.


### `transit_details` (1:1 k `segments`)

| Sloupec | Typ | Popis |
|---|---|---|
| `segment_id` | UUID, PK, FK → segments | ON DELETE CASCADE |
| `transport_mode` | transport_mode, NOT NULL | |
| `provider_id` | UUID, FK → robotaxi_providers, nullable | Poskytovatel robotaxi; u `transport_mode = robotaxi` povinné; u ostatních režimů `NULL`; ON DELETE RESTRICT |
| `distance_meters` | INTEGER, nullable | Délka trasy v metrech; pokud vyplněno, musí být `>= 0` |
| `booking_reference` | VARCHAR, nullable | ID rezervace z API |
| `pickup_zone_place_id` | UUID, FK → places, nullable | Pickup zóna robotaxi; ON DELETE RESTRICT; aplikace validuje kategorii `robotaxi_pickup_zone`; jen u `transport_mode = robotaxi`, jinak `NULL` |
| `dropoff_zone_place_id` | UUID, FK → places, nullable | Dropoff zóna robotaxi; ON DELETE RESTRICT; aplikace validuje kategorii `robotaxi_pickup_zone`; jen u `transport_mode = robotaxi`, jinak `NULL` |
| `estimated_wait_minutes` | SMALLINT, nullable | Odhad čekání na přistavení vozu v minutách; `>= 0`; jen u `transport_mode = robotaxi`, jinak `NULL` |
| `passenger_count` | SMALLINT, nullable | Počet cestujících; `>= 1`; jen u `transport_mode = robotaxi`, jinak `NULL` |
| `vehicle_model_id` | UUID, FK → robotaxi_vehicle_models, nullable | Model vozu; ON DELETE RESTRICT; jen u `transport_mode = robotaxi`, jinak `NULL`; aplikace validuje shodu `vehicle_model.provider_id` s `provider_id` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |


---

## Poznámky k implementaci

### Sémantika segmentů

Primární zdroj pravdy pro typ segmentu je sloupec `segment_kind`. Aplikace ho nastaví při INSERT/UPDATE a validuje konzistenci s `end_place_id`, `transit_details`, kategorií místa, `recommended_age_*` a `difficulty`.

**Matice konzistence** (aplikace validuje při zápisu):

| `segment_kind` | `end_place_id` | `transit_details` | Kategorie `start_place_id` | `recommended_age_*` | `difficulty` |
|---|---|---|---|---|---|
| `accommodation` | NULL | nesmí existovat | `accommodation` | vždy NULL | vždy NULL |
| `activity` | NULL nebo NOT NULL | nesmí existovat | libovolná (včetně `accommodation` — např. spa v hotelu) | volitelné | volitelné (vstup pro agregaci `max_difficulty`) |
| `transit` | NOT NULL | povinný řádek | libovolná | vždy NULL | volitelné (jen pro `clothing_rules`, ne pro `max_difficulty`) |

**`end_place_id` u transitů:** `start_place_id = end_place_id` je **povolen** — okružní jízda (vyhlídková loď, scenic drive robotaxi) reálně existuje a končí tam, kde začala. Na rozdíl od aktivit se rovnost nevaliduje jako chyba; lookup počasí pak oba směry vyhodnotí jako stejný region (deduplikace `(weather_region_id, week_start)` to pokrývá).

**Validace `end_place_id` u aktivit** (aplikace při zápisu):
- aktivita na jednom místě → `end_place_id` musí být NULL
- aktivita končí jinde než začíná → `end_place_id` NOT NULL a `end_place_id ≠ start_place_id`
- `start_place_id = end_place_id` → chyba (místo NULL)

**Hraniční případy transit uzlů:**
- `robotaxi_pickup_zone` — cíl chůze/robotaxi před nástupem; statický segment na pickup zóně (např. čekání) = `segment_kind = activity` (vzácné)
- `airport` — cíl transit segmentu s `transport_mode = plane` (check-in, pickup)
- `parking_lot` — cíl transit segmentu s `own_car` / `car_rental` (vyzvednutí/vrácení auta); také typický `robotaxi_access_place_id` (konec silnice / trailhead)
- `public_transport_terminal` — vlak, metro, autobus (letiště nepatří sem)

#### Skladba last-mile podle `places.robotaxi_access`

Kurátorovaná last-mile metadata na cílovém místě (viz [Places — last-mile robotaxi](places.md#places)) doporučují skladbu transit segmentů. Schema je **negeneruje** — autor / aplikace je sestaví ručně nebo z UI šablony.

| `robotaxi_access` cíle | Doporučená skladba |
|---|---|
| `direct` | Jeden `transit` + `robotaxi` s `end_place_id` = cílové místo |
| `via_access_point` | `transit` + `robotaxi` s `end_place_id` = `robotaxi_access_place_id`, pak `transit` + `walk` z access pointu na cíl (`distance_meters` ≈ `robotaxi_approach_walk_meters`) |
| `not_accessible` | Robotaxi segment k cíli nedoporučovat; jiný `transport_mode` nebo chůze od jiného uzlu |
| `NULL` | Last-mile neznámé — žádná šablona; service-area soft varování stále platí |

`dropoff_zone_place_id` v `transit_details` zůstává volitelný uzel providera (kategorie `robotaxi_pickup_zone`); `robotaxi_access_place_id` na cílovém POI je kurátorovaný „konec silnice“ pro last-mile — můžou se shodovat, ale nejsou totéž.

**Ubytování na více nocí:** nepoužívej jeden dlouhý `accommodation` segment přes celý pobyt, pokud by překrýval denní program. Pobyt rozděl na nepřekrývající se ubytovací úseky — typicky check-in blok, noční úseky a/nebo check-out blok podle toho, co má být v timeline vidět. Cena jedné rezervace se uloží jen jednou (např. na první ubytovací úsek) nebo se deterministicky rozdělí mezi úseky, aby `trips.total_cost_usd` a `trips.total_cost_home` nebyly nadhodnocené — split musí platit současně pro `local_price_amount`, `home_price_amount` a `price_usd`. Frontend může úseky vizuálně seskupit jako jeden hotelový pobyt; DB timeline zůstává striktně lineární. `difficulty` u ubytování je vždy `NULL`.

Segmenty se modelují jako polootevřené intervaly `[start_time, end_time)`. Příklad timeline (ubytovací úsek končí v 10:00, chůze začíná v 10:00) je validní — dotyk na hranici není překryv.

Timeline segmentů načítej `ORDER BY start_time, id`. Sloupec `id` slouží jen jako deterministický tie-breaker pro řazení (např. po migraci dat), ne jako podpora paralelních segmentů.

Nepřekrývání segmentů vynucuje DB exclusion constraint (aplikace validuje navíc před zápisem kvůli lepší chybové hlášce); konzistenci `transit_details` vynucuj v aplikační vrstvě — viz [DB constrainty](constraints-and-indexes.md#db-constrainty).

### Obsah segmentu

Autorský obsah segmentu (`title`, `description`, `segment_images`) je oddělený od strukturálních dat a neagreguje se na `trips`. Platí pro všechny `segment_kind`.

**Nadpis:** `segments.title` je volitelný. Pokud je `NULL` (nebo prázdný po normalizaci), FE zobrazí `places.name` u `start_place_id`. Pokud je i `places.name` `NULL`, FE zobrazí generický placeholder.

**Popis:** `segments.description` je volitelný; zobrazí se v detailu segmentu na timeline.

**Galerie:** Obrázky načítej z `segment_images` pro daný `segment_id`, seřazené `ORDER BY sort_order, id`. Každý řádek má `image_url` (veřejná URL — upload a storage mimo DB, stejně jako u `trips.cover_image_url`) a volitelný `caption`. Segment může mít 0..N obrázků.

**Timeline thumbnail:** Denormalizovaný náhled na segmentu neexistuje — FE volitelně použije první obrázek dle `sort_order, id` jako miniaturu v timeline.

**Validace (aplikace):** `image_url` musí být neprázdná HTTPS URL; prázdné `title` / `description` / `caption` normalizuj na `NULL`; max. počet obrázků per segment volitelně v aplikaci (např. 20).

**Příklad dotazu pro detail itineráře:**

```sql
SELECT s.*, sp.name AS start_place_name
FROM segments s
JOIN places sp ON sp.id = s.start_place_id
WHERE s.trip_id = :trip_id
ORDER BY s.start_time, s.id;

-- Galerie per segment (0..N řádků):
SELECT * FROM segment_images
WHERE segment_id = :segment_id
ORDER BY sort_order, id;
```

**Migrace (SQL návrh):**

```sql
ALTER TABLE segments
  ADD COLUMN title VARCHAR,
  ADD COLUMN description TEXT;

CREATE TABLE segment_images (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  segment_id UUID NOT NULL REFERENCES segments(id) ON DELETE CASCADE,
  image_url TEXT NOT NULL,
  caption TEXT,
  sort_order SMALLINT NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_segment_images_segment_sort
  ON segment_images (segment_id, sort_order);
```

(+ standardní `updated_at` trigger na `segment_images`, konzistentně s ostatními tabulkami.)

### `segment_difficulty`

Referenční popisy níže patří do FE locale souborů, ne do PostgreSQL.

| Hodnota | FE i18n (příklad) | Příklad |
|---|---|---|
| `easy` | Nízká | Procházka po rovině, muzeum, kavárna |
| `moderate` | Střední | Celodenní prohlídka města, lehký výstup |
| `hard` | Vyšší | Túra s převýšením, celodenní cyklistika |
| `extreme` | Náročná | Via ferrata, alpský trek, náročné sportovní aktivity |

### Cena a délka výletu

**Segmenty (zdroj pravdy):**
- **Místní cena (destinace):** `local_price_amount`, `local_price_currency` — co platíte na místě
- **Domácí/plánovací cena:** `home_price_amount`, `exchange_rate_local_to_home` — přepočet do `trips.home_currency`
- **Katalogová cena (USD kanon):** `price_usd`, `exchange_rate_local_to_usd` — srovnání a filtry veřejného katalogu
- U `segment_kind = accommodation` je `local_price_amount` cena přiřazená konkrétnímu ubytovacímu úseku; u jedné rezervace rozdělené do více úseků se cena uloží jen jednou nebo se rozpočítá, aby se v `total_cost_usd` a `total_cost_home` nezapočetla vícekrát
- Délka per segment: `end_time - start_time`

**Trips (cache):**
- `home_currency` — měna, ve které autor chce výlet plánovat a zobrazovat rozpočet (editovatelná; výchozí z `users.home_currency` při vytvoření výletu)
- `total_cost_home` — agregovaný rozpočet v `home_currency` (`SUM home_price_amount`); pro editor a detail výletu
- `total_cost_usd` — agregovaná cena v USD (`SUM price_usd`); **jediný zdroj pro filtrování a řazení veřejného katalogu**
- `total_duration_minutes` — viz [Agregace na úrovni `trips`](implementation-notes.md#agregace-na-úrovni-trips)

Autor výletu cache ceny nevyplňuje ručně; aplikace je přepočítá při každé změně segmentů nebo `home_currency`. Orientační přepočet ceny pro prohlížeče s jinou preferovanou měnou (mimo `home_currency` výletu) probíhá v UI za běhu z `price_usd` — bez per-user cache v DB.

#### Kurz měny

Při INSERT/UPDATE segmentu aplikace přepočítá obě konverzní vrstvy (v jedné transakci s agregací na `trips`):

1. Uživatel zadá `local_price_amount` + `local_price_currency`.
2. **USD vrstva:** pro `local_price_currency = 'USD'`: `exchange_rate_local_to_usd = 1.0`; jinak načíst kurz `local → USD` z externího API (ECB, OpenExchangeRates apod.) s fallbackem na aplikační cache; pokud API i cache prázdné → **chyba validace**. Uložit kurz a spočítat `price_usd = ROUND(local_price_amount × exchange_rate_local_to_usd, 2)`.
3. **Home vrstva:** načíst `home_currency` z `trips`. Pokud `local_price_currency = home_currency`: `exchange_rate_local_to_home = 1.0`, `home_price_amount = local_price_amount`; jinak načíst kurz `local → home_currency` (stejný provider/fallback jako u USD). Uložit kurz a spočítat `home_price_amount = ROUND(local_price_amount × exchange_rate_local_to_home, 2)`.
4. Obě konverze jdou **přímo z místní měny** (ne přes USD), aby se minimalizovalo dvojité zaokrouhlování.
5. Agregovat `total_cost_usd = SUM(price_usd)` a `total_cost_home = SUM(home_price_amount)`.

**Změna `trips.home_currency`:** v jedné transakci přepočítat `exchange_rate_local_to_home` a `home_price_amount` na **všech** segmentech výletu, poté `total_cost_home`. USD vrstva (`price_usd`, `total_cost_usd`) se nemění.

**Bezplatné segmenty (`local_price_amount = 0`):** uložit `local_price_currency = 'USD'`, `exchange_rate_local_to_usd = 1.0`, `exchange_rate_local_to_home = 1.0`, `price_usd = 0`, `home_price_amount = 0`. Jde o auditní konvenci — UI zobrazuje „zdarma“ bez ohledu na měnu.

**Katalogové dotazy** (vždy USD — `total_cost_home` se v katalogu nepoužívá):

**Pravidlo filtrovaného katalogu:** U dotazů s filtrem na teplotu nebo náročnost musí být `feels_like_min_c`, `feels_like_max_c` resp. `max_difficulty` **NOT NULL** — výlet bez vyplněné cache se nezobrazí. Primární teplotní filtr je **pocitová** teplota; vzduchová `temp_*` zůstává pro zobrazení a volitelný sekundární filtr. U věku naopak `NULL` znamená „bez limitu“ (výlet vhodný pro všechny věky).

```sql
-- Filtrování dle max. ceny
SELECT * FROM trips
WHERE status = 'published' AND visibility = 'public'
  AND total_cost_usd <= :max_cena_usd;

-- Filtrování dle max. délky programu (minuty) — ne dojezd z místa
SELECT * FROM trips
WHERE status = 'published' AND visibility = 'public'
  AND total_duration_minutes <= :max_minut;

-- Filtrování dle dojezdu z vybraného místa — viz travel-times.md
SELECT t.*
FROM trips t
LEFT JOIN travel_time_estimates e
  ON e.destination_place_id = t.destination_place_id
 AND e.origin_place_id = :origin_place_id
 AND e.transport_mode = t.outbound_transport_mode
WHERE t.status = 'published' AND t.visibility = 'public'
  AND t.destination_place_id IS NOT NULL
  AND t.outbound_transport_mode IS NOT NULL
  AND (
    t.destination_place_id = :origin_place_id
    OR (e.id IS NOT NULL AND e.duration_minutes <= :max_dojezd_minut)
  );

-- Kombinovaný filtr (cena + délka programu + věk + náročnost + pocitová teplota)
SELECT * FROM trips
WHERE status = 'published' AND visibility = 'public'
  AND total_cost_usd <= :max_cena_usd
  AND total_duration_minutes <= :max_minut
  AND (recommended_age_min IS NULL OR recommended_age_min <= :vek)
  AND (recommended_age_max IS NULL OR recommended_age_max >= :vek)
  AND max_difficulty IS NOT NULL AND max_difficulty <= :max_narocnost
  AND feels_like_min_c IS NOT NULL AND feels_like_min_c >= :min_teplota
  AND feels_like_max_c IS NOT NULL AND feels_like_max_c <= :max_teplota
ORDER BY total_cost_usd ASC;
```

Filtr dojezdu (origin place + max. minuty) je podrobně v [Katalogový filtr dojezdu](travel-times.md#katalogový-filtr-dojezdu); lze ho kombinovat s výše uvedenými predikáty přes `JOIN` / `LEFT JOIN` na `travel_time_estimates`.


### Doporučený věk a náročnost

**Segmenty (zdroj pravdy):** Sloupce `recommended_age_min` a `recommended_age_max` se vyplňují u aktivit (`segment_kind = activity`). U dopravy a ubytování zůstávají `NULL`.

**Trips (cache pro katalog):** Sloupce `recommended_age_min`, `recommended_age_max` a `max_difficulty` jsou agregovaná cache odvozená ze segmentů — viz [Agregace na úrovni `trips`](implementation-notes.md#agregace-na-úrovni-trips). Autor výletu je nevyplňuje ručně; aplikace je přepočítá při každé změně segmentů.

`difficulty` u aktivit a dopravních segmentů (`segment_kind = transit`) se vyplňuje **explicitně** (volitelné) — primárně pro vyhodnocení `clothing_rules` na úrovni segmentu. U ubytování (`segment_kind = accommodation`) zůstává `difficulty` vždy `NULL`. Do agregace `trips.max_difficulty` se započítávají **jen aktivity** — nejnáročnější aktivita určuje náročnost celého výletu.

Validace: rozsah věku a vztah min/max vynucuje DB CHECK constraint na `segments` i `trips`; aplikace validuje před zápisem pro lepší chybové hlášky. Pokud agregace aktivit vede k neslučitelnému rozsahu (např. aktivita A `min 12`, aktivita B `max 8` → cache `min 12, max 8`), aplikace detekuje konflikt **při uložení segmentu** (ne až při publikaci) a zobrazí chybu autorovi — výlet nemá smysl pro žádný jednotlivý věk. Bez této validace by INSERT/UPDATE selhal na DB CHECK `recommended_age_min <= recommended_age_max` na `trips`.

**Katalogový dotaz** (filtrování výletů vhodných pro daný věk):

```sql
SELECT *
FROM trips
WHERE status = 'published'
  AND visibility = 'public'
  AND (recommended_age_min IS NULL OR recommended_age_min <= :vek)
  AND (recommended_age_max IS NULL OR recommended_age_max >= :vek);
```

**Katalogový dotaz** (filtrování dle max. náročnosti — enum musí být deklarován v pořadí rostoucí náročnosti; `NULL` = výlet bez vyplněné náročnosti, ve filtrovaném katalogu se nezobrazí):

```sql
SELECT *
FROM trips
WHERE status = 'published'
  AND visibility = 'public'
  AND max_difficulty IS NOT NULL AND max_difficulty <= :max_narocnost;
```

Pro kombinované filtry s cenou, délkou a teplotou viz [Cena a délka výletu](#cena-a-délka-výletu), [Teplota výletu](weather-and-climate.md#teplota-výletu) a [Teplota mořské vody](weather-and-climate.md#teplota-mořské-vody-sst).

### `transit_details` — pravidla

Tabulka `transit_details` existuje pouze u segmentů s `segment_kind = transit`. V aplikační vrstvě vynucuj (bez DB triggerů):

- INSERT do `transit_details` jen u segmentů s `segment_kind = transit`
- segmenty `accommodation` a `activity` nemají `transit_details`
- `transport_mode = 'robotaxi'` ⇒ `provider_id` NOT NULL (FK na aktivního `robotaxi_providers`)
- ostatní `transport_mode` ⇒ `provider_id`, `pickup_zone_place_id`, `dropoff_zone_place_id`, `estimated_wait_minutes`, `passenger_count`, `vehicle_model_id` vždy `NULL`
- u robotaxi: `pickup_zone_place_id` / `dropoff_zone_place_id` volitelné, ale pokud vyplněné, musí být `places` s kategorií `robotaxi_pickup_zone`
- u robotaxi: pokud je `vehicle_model_id` vyplněné, `robotaxi_vehicle_models.provider_id` musí odpovídat `provider_id`
- u robotaxi: pokud jsou vyplněné `passenger_count` i `vehicle_model_id`, `passenger_count <= seat_count`

