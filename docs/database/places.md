# Places

[← Přehled schématu](README.md)

Pravidla zápisu a agregace uživatelských recenzí (trip i place) — viz [Recenze](users-and-trips.md#recenze).

## Tabulky

### `place_categories`

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `name` | VARCHAR, UNIQUE | Slug key (e.g. `accommodation`) |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Seed data:**

| slug | typická použití | poznámka |
|---|---|---|
| `accommodation` | Hotely, hostely, apartmány | Povinná kategorie pro `segment_kind = accommodation` na `start_place_id` |
| `landmark` | Památky, hrady, mosty, náměstí | Venkovní historické body zájmu |
| `restaurant_dining` | Restaurace, kavárny, bary | Gastronomické aktivity |
| `entertainment_sports` | Kina, stadiony, zábavní parky, sportoviště | |
| `viewpoint` | Vyhlídky, rozhledny, scenic spots | |
| `charging_fueling_hub` | Nabíjecí stanice, čerpací stanice | Infrastruktura pro EV a robotaxi |
| `public_transport_terminal` | Nádraží, autobusová nádraží, stanice metra/tramvaje | Vlak, metro, autobus — letiště nepatří sem |
| `nature_park` | Národní parky, rezervace, lesní stezky | Turistika, hiking |
| `robotaxi_pickup_zone` | Pickup zóny pro autonomní taxi | Typický cíl `transit` segmentu před nástupem |
| `culture_museum` | Muzea, galerie, výstavní prostory | Odděleno od `landmark` (venkovní památky) |
| `shopping_retail` | Obchody, nákupní centra, trhy | Aktivity mimo gastronomii |
| `beach_waterfront` | Pláže, jezera, nábřeží, koupaliště | Jiné balení než `nature_park` (turistika) |
| `wellness_spa` | Samostatná spa, lázně mimo hotel | Spa v hotelu = `activity` na `start_place_id` s kategorií `accommodation` |
| `airport` | Letiště | Typický cíl `transit` segmentu s `transport_mode = plane` |
| `parking_lot` | Parkoviště, P+R, car rental pickup | Typický cíl `transit` u `own_car` / `car_rental` |


### `places`

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `name` | VARCHAR, nullable | Jednojazyčný název z externího API nebo adminu (faktický obsah, ne katalog překladů); prázdný řetězec normalizuj na `NULL` |
| `description` | TEXT, nullable | Krátký popis místa (Google Maps–style); prázdný řetězec normalizuj na `NULL` |
| `website_url` | TEXT, nullable | Webová stránka místa; při vyplnění HTTPS URL (stejný vzor jako `robotaxi_providers.website_url`) |
| `address` | TEXT, nullable | Volný text adresy; prázdný řetězec normalizuj na `NULL` |
| `phone_calling_code` | VARCHAR, nullable | Mezinárodní telefonní předvolba bez `+` (např. `420`, `1`, `43`) — **ne** ISO `country_code`; jen číslice, typicky 1–3 znaky |
| `telephone` | VARCHAR, nullable | Národní / účastnické číslo bez předvolby (např. `222111222`); FE skládá zobrazení `+{phone_calling_code} {telephone}` |
| `accepted_payments` | place_accepted_payments, nullable | Jak se na místě platí: `card` / `cash` / `card_and_cash`; `NULL` = neznámé / nezadáno — UI nic nezobrazí. Bez stavu „nebere nic“ (v1); bezplatné / předplacené místo nech `NULL` nebo doplň později. Žádná agregace na `trips`, žádný katalog / junction. |
| `rating` | NUMERIC(2, 1), nullable | **Externí** hodnocení ve stylu Google Maps (`1.0`–`5.0`) z importu / adminu — **ne** agregace z `place_reviews`; `NULL` = neznámé |
| `review_rating_avg` | NUMERIC(2, 1), nullable | Průměrné uživatelské hodnocení místa (`1.0`–`5.0`); cache z `place_reviews.score`; ne ručně editovatelná; `NULL` = žádné recenze (`review_rating_count = 0`) |
| `review_rating_count` | INTEGER, NOT NULL | Počet uživatelských recenzí; cache z `place_reviews`; ne ručně editovatelná; výchozí `0` |
| `latitude` | NUMERIC(9, 6), NOT NULL | Zeměpisná šířka (`-90` až `90`) |
| `longitude` | NUMERIC(9, 6), NOT NULL | Zeměpisná délka (`-180` až `180`) |
| `category_id` | UUID, FK → place_categories | ON DELETE RESTRICT |
| `weather_region_id` | UUID, FK → weather_regions, nullable | ON DELETE SET NULL |
| `external_source` | VARCHAR, nullable | Zdroj importu (např. `google_maps`); `NULL` = místo založené ručně v adminu |
| `external_place_id` | VARCHAR, nullable | Identifikátor místa u daného zdroje; párový s `external_source` (buď obě `NULL`, nebo obě vyplněné) |
| `country_code` | CHAR(2), nullable | ISO 3166-1 alpha-2 (`CZ`, `AT`, …); `NULL` = neznámá země — geo pravidla cestovních požadavků se pro dané místo přeskočí |
| `postal_code` | VARCHAR, nullable | Normalizované PSČ/ZIP z geocodingu; pro auto-párování s `weather_regions` |
| `subdivision_code` | VARCHAR, nullable | ISO 3166-2 bez prefixu země (`CA`, `20`) |
| `subdivision_name` | VARCHAR, nullable | Lidsky čitelný název subdivize |
| `locality` | VARCHAR, nullable | Město / obec z geocodingu |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Přijímané platby (`accepted_payments`) — sémantika pro FE:**

| Hodnota | Význam | UI |
|---|---|---|
| `NULL` | Neznámé / nezadáno | Žádný platební blok |
| `card` | Jen platba kartou | Chip „Platba kartou“ |
| `cash` | Jen hotovost | Chip „Hotovost“ |
| `card_and_cash` | Karta i hotovost | Oba chipy, nebo jeden „Karta i hotovost“ (volba FE; DB drží jednu hodnotu) |

Labely z FE i18n podle enum hodnoty + `users.locale`.

### `place_reviews`

Uživatelská recenze místa — skóre, text a volitelná galerie médií. Jeden aktivní hlas na uživatele a místo. Agregace do `places.review_rating_avg` / `places.review_rating_count` (viz [Recenze](users-and-trips.md#recenze)). Oddělené od externího `places.rating` (Google Maps–style import).

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `place_id` | UUID, FK → places | ON DELETE CASCADE |
| `user_id` | UUID, FK → users | ON DELETE RESTRICT; autor recenze |
| `score` | SMALLINT, NOT NULL | Celé hodnocení `1`–`5` |
| `body` | TEXT, nullable | Text recenze v jazyce autora (UGC, ne systémový překlad); prázdný řetězec normalizuj na `NULL` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**UNIQUE** `(place_id, user_id)` — upsert při změně skóre / textu.

### `place_review_media`

Galerie médií k recenzi místa — 0..N řádků na recenzi. Stejný tvar a URL pattern jako `trip_review_media`.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `review_id` | UUID, FK → place_reviews | ON DELETE CASCADE |
| `media_kind` | review_media_kind, NOT NULL | `image` nebo `video` |
| `media_url` | TEXT, NOT NULL | Veřejná HTTPS URL (CDN/R2/S3) |
| `poster_url` | TEXT, nullable | Náhledový obrázek u videa (HTTPS); u `media_kind = image` vždy `NULL` |
| `caption` | TEXT, nullable | Volitelný popisek v jazyce autora; prázdný řetězec normalizuj na `NULL` |
| `sort_order` | SMALLINT, NOT NULL | Pořadí v galerii; výchozí `0`; `>= 0`; řazení `ORDER BY sort_order, id` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

Validace v aplikaci: stejná jako u `trip_review_media` — viz [Recenze](users-and-trips.md#recenze).


---

## Poznámky k implementaci

### Import a deduplikace míst

Dvojice `(external_source, external_place_id)` je **stabilní identita importovaného místa**. Bez ní by opakovaný import založil duplicitní řádky, `place_reviews` by se rozpadly mezi dva záznamy téhož místa a `places.rating` by přestal odpovídat realitě.

- Partial unique index `uq_places_external` (`WHERE external_source IS NOT NULL`) — ručně založená místa mají obě pole `NULL` a index je neomezuje.
- CHECK párovosti: `(external_source IS NULL) = (external_place_id IS NULL)` — nelze uložit ID bez zdroje ani naopak.
- Re-import je **upsert** na tuto dvojici:

```sql
INSERT INTO places (external_source, external_place_id, name, description, latitude, longitude, rating, ...)
VALUES (:source, :external_id, ...)
ON CONFLICT (external_source, external_place_id) DO UPDATE SET
  name = EXCLUDED.name,
  description = EXCLUDED.description,
  latitude = EXCLUDED.latitude,
  longitude = EXCLUDED.longitude,
  rating = EXCLUDED.rating,
  updated_at = now();
```

- Import **nikdy** nepřepisuje `review_rating_avg` / `review_rating_count` (cache z UGC `place_reviews`), `weather_region_id` (ruční / admin přiřazení) ani `category_id`, pokud ho admin ručně změnil.
- `external_source` je volný slug zdroje, ne enum — přidání dalšího poskytovatele nevyžaduje migraci typu.
- Ručně založené místo lze později „adoptovat“ doplněním obou polí; kolizi s již importovaným místem zachytí unique index.

### Geografické dotazy

Index `idx_places_coordinates` postačí pro jednoduché bounding-box dotazy ve v1. Pro produkční proximity search (radius kolem bodu) je v backlogu **PostGIS v2** — sloupec `geom geography(POINT, 4326)` na `places` a GIST index.

Auto-párování `weather_regions` probíhá administrativně (ISO kódy na `places` a `weather_regions`), ne geometrickým poloměrem. Poslední fallback: nejbližší `center_latitude` / `center_longitude` mezi regiony **stejné země** (`country_code`); finální rozhodnutí vždy ruční `places.weather_region_id`. Ruční přiřazení nesmí obejít invariant stejné země a má respektovat jemnější shodu `postal_code` / `locality` / `subdivision_code`, pokud jsou na místě známé. Párování volí nejjemnější oblast; když pro ni chybí týdenní data, lookup za běhu zkusí rodiče — viz [Hierarchie oblastí a geo-fallback](weather-and-climate.md#hierarchie-oblastí-a-geo-fallback).

