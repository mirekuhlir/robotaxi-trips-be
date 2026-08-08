# Robotaxi

[← Přehled schématu](README.md)

## Tabulky

### `robotaxi_providers`

Katalog poskytovatelů autonomní taxi — nahrazuje enum `transit_provider`. Nové providery lze přidávat bez migrace enumu.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `slug` | VARCHAR, UNIQUE, NOT NULL | Stabilní klíč, např. `waymo`, `tesla`, `zoox` |
| `name` | VARCHAR, NOT NULL | Zobrazovaný název (`Waymo`, `Tesla`, …) |
| `website_url` | TEXT, nullable | Web providera |
| `app_deeplink_url` | TEXT, nullable | Deep-link do mobilní appky providera |
| `is_active` | BOOLEAN | Výchozí `true` — skrytí bez mazání |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Seed data:**

| slug | name |
|---|---|
| `waymo` | Waymo |
| `tesla` | Tesla |
| `zoox` | Zoox |
| `cruise` | Cruise |
| `apollo_go` | Baidu Apollo Go |
| `weride` | WeRide |
| `pony_ai` | Pony.ai |
| `motional` | Motional |
| `other` | Other |

### `robotaxi_vehicle_models`

Katalog modelů vozidel per provider — kapacita, bezbariérovost, dětská sedačka.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `provider_id` | UUID, FK → robotaxi_providers | ON DELETE CASCADE |
| `slug` | VARCHAR, NOT NULL | Stabilní klíč per provider, např. `jaguar_i_pace` |
| `name` | VARCHAR, NOT NULL | Zobrazovaný název (`Jaguar I-Pace`, `Zoox pod`, …) |
| `seat_count` | SMALLINT, NOT NULL | Počet sedadel; `>= 1` |
| `wheelchair_accessible` | BOOLEAN | Výchozí `false` |
| `child_seat_available` | BOOLEAN | Výchozí `false` |
| `is_active` | BOOLEAN | Výchozí `true` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Omezení:** `UNIQUE (provider_id, slug)`.

### `provider_service_areas`

Servisní oblasti providerů jsou ve v1 administrativní a orientační katalog na úrovni locality. Nejde o geometrickou geofence a shoda sama nepotvrzuje, že provider obslouží konkrétní pickup/dropoff bod. Doména je oddělená od `weather_regions`.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `provider_id` | UUID, FK → robotaxi_providers | ON DELETE CASCADE |
| `name` | VARCHAR, NOT NULL | Zobrazovaný název oblasti (`San Francisco`, `Phoenix`, …) |
| `country_code` | CHAR(2), NOT NULL | ISO 3166-1 alpha-2 |
| `subdivision_code` | VARCHAR, NOT NULL | ISO 3166-2 bez prefixu země |
| `subdivision_name` | VARCHAR, nullable | Lidsky čitelný název subdivize |
| `locality` | VARCHAR, NOT NULL | Město / obec |
| `status` | robotaxi_service_status | Výchozí `waitlist` |
| `launched_at` | DATE, nullable | Datum spuštění veřejného provozu |
| `operates_24_7` | BOOLEAN | Výchozí `true` |
| `daily_start_local` | TIME, nullable | Začátek denního provozu (lokální čas oblasti); povinné když `operates_24_7 = false` |
| `daily_end_local` | TIME, nullable | Konec denního provozu; povinné když `operates_24_7 = false`; hodnota **nižší** než `daily_start_local` znamená okno přes půlnoc (viz [Provozní okno](#provozní-okno)) |
| `serves_airport` | BOOLEAN | Výchozí `false` — obsluha letišť |
| `allows_highway` | BOOLEAN | Výchozí `false` — jízda po dálnici |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Omezení:** `UNIQUE (provider_id, country_code, subdivision_code, locality)`.

**Backlog v2:** polygonová geofence přes PostGIS (`geom geography(POLYGON, 4326)`) pro skutečný point-in-polygon test pickup/dropoff bodů.

### `robotaxi_advisory_items`

Katalog robotaxi upozornění (počasí × provoz) — oddělený od `clothing_items`.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `slug` | VARCHAR, UNIQUE, NOT NULL | Stabilní klíč, např. `dense_fog_disruption`, `storm_disruption` |
| `severity` | robotaxi_advisory_severity, NOT NULL | `info` nebo `warning` |
| `sort_order` | SMALLINT, NOT NULL | Pořadí v seznamu; výchozí `0`; `>= 0` |
| `is_active` | BOOLEAN | Výchozí `true` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Seed data:**

| slug | severity | sort_order |
|---|---|---|
| `dense_fog_disruption` | `warning` | 10 |
| `storm_disruption` | `warning` | 20 |

### `robotaxi_advisory_rules`

Pravidla mapující počasí na robotaxi upozornění. Vyhodnocují se **jen pro segmenty** `segment_kind = transit` s `transport_mode = robotaxi`. Všechny podmínky jsou relační — skalární sloupce nebo řádky v junction tabulkách. Vyplněné podmínky se kombinují logikou **AND**; u junction tabulek platí logika **IN**.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `advisory_item_id` | UUID, FK → robotaxi_advisory_items | ON DELETE CASCADE |
| `priority` | SMALLINT | Vyšší = důležitější v UI (výchozí `0`) |
| `visibility_avg_m_lte` | INTEGER, nullable | Match proti `weather_records.visibility_avg_m` |
| `is_active` | BOOLEAN | Výchozí `true` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Validace pravidla (aplikace):** musí existovat alespoň jedna aktivní podmínka — jinak match-all riziko:
- alespoň jeden skalární sloupec NOT NULL, nebo
- ≥1 řádek v libovolné junction tabulce níže

**Příklad seed pravidel:**

| Položka | `robotaxi_advisory_rules` | Junction tabulky |
|---|---|---|
| `dense_fog_disruption` — mlha | — | `fog`, `dense_fog` v `robotaxi_advisory_rule_fog_conditions` |
| `storm_disruption` — bouřka | — | `storm` v `robotaxi_advisory_rule_precipitation_intensities`; `gale`, `storm` v `robotaxi_advisory_rule_wind_forces` |

### `robotaxi_advisory_rule_fog_conditions`

Množinová podmínka na mlhu (IN logika).

| Sloupec | Typ | Popis |
|---|---|---|
| `advisory_rule_id` | UUID, PK, FK → robotaxi_advisory_rules | ON DELETE CASCADE |
| `fog_condition` | fog_condition, PK | |

### `robotaxi_advisory_rule_precipitation_intensities`

Množinová podmínka na intenzitu srážek (IN logika).

| Sloupec | Typ | Popis |
|---|---|---|
| `advisory_rule_id` | UUID, PK, FK → robotaxi_advisory_rules | ON DELETE CASCADE |
| `precipitation_intensity` | precipitation_intensity, PK | |

### `robotaxi_advisory_rule_wind_forces`

Množinová podmínka na sílu větru (IN logika).

| Sloupec | Typ | Popis |
|---|---|---|
| `advisory_rule_id` | UUID, PK, FK → robotaxi_advisory_rules | ON DELETE CASCADE |
| `wind_force` | wind_force, PK | |

### `segment_robotaxi_advisories`

Cache robotaxi upozornění per segment (0..N položek). Přepočet v aplikaci — viz [Robotaxi upozornění](#robotaxi-upozornění).

| Sloupec | Typ | Popis |
|---|---|---|
| `segment_id` | UUID, PK, FK → segments | ON DELETE CASCADE |
| `advisory_item_id` | UUID, PK, FK → robotaxi_advisory_items | ON DELETE RESTRICT |
| `priority` | SMALLINT, NOT NULL | Nejvyšší priorita matching `robotaxi_advisory_rules` pro daný segment a položku |

### `trip_robotaxi_advisories`

Agregovaná cache robotaxi upozornění na úrovni výletu (0..N položek). Při sloučení ze segmentů platí u stejné položky **nejvyšší** `priority`.

| Sloupec | Typ | Popis |
|---|---|---|
| `trip_id` | UUID, PK, FK → trips | ON DELETE CASCADE |
| `advisory_item_id` | UUID, PK, FK → robotaxi_advisory_items | ON DELETE RESTRICT |
| `priority` | SMALLINT, NOT NULL | Max priority ze `segment_robotaxi_advisories` |


---

## Poznámky k implementaci

### Robotaxi upozornění

Robotaxi upozornění (počasí × provoz) jsou **oddělená doména** od weather packingu i cestovních požadavků. Cache je v relačních tabulkách `segment_robotaxi_advisories` (per segment) a `trip_robotaxi_advisories` (agregace na výlet) — UI labely mapuje FE z i18n klíčů odvozených od `robotaxi_advisory_items.slug` a `severity`.

#### Cache tabulky

| Tabulka | Úroveň | Obsah |
|---|---|---|
| `segment_robotaxi_advisories` | segment | Které `robotaxi_advisory_items` doporučit pro daný robotaxi segment + nejvyšší `priority` z matching pravidel |
| `trip_robotaxi_advisories` | výlet | Agregovaný seznam s `priority` (union ze segmentů; u stejné položky nejvyšší priorita) |
| `trips.robotaxi_advisories_computed_at` | výlet | Čas posledního úspěšného přepočtu; `NULL` = nepočítáno nebo invalidováno |

**Příklad dotazu — upozornění pro detail výletu:**

```sql
SELECT rai.slug, rai.severity, tra.priority
FROM trip_robotaxi_advisories tra
JOIN robotaxi_advisory_items rai ON rai.id = tra.advisory_item_id
WHERE tra.trip_id = :trip_id
ORDER BY tra.priority DESC, rai.sort_order;
```

**Příklad dotazu — upozornění per segment (timeline UI):**

```sql
SELECT rai.slug, rai.severity, sra.priority
FROM segment_robotaxi_advisories sra
JOIN robotaxi_advisory_items rai ON rai.id = sra.advisory_item_id
WHERE sra.segment_id = :segment_id
ORDER BY sra.priority DESC, rai.sort_order;
```

#### Výpočet na segment

1. Přeskočit segmenty, které nejsou `segment_kind = transit` s `transport_mode = robotaxi`.
2. Načíst `weather_records` pro segment podle [Lookup počasí podle typu segmentu](weather-and-climate.md#lookup-počasí-podle-typu-segmentu) (kroky 1–3; včetně [geo-fallbacku](weather-and-climate.md#hierarchie-oblastí-a-geo-fallback) na rodičovskou oblast). Klima se pro advisories nepoužívá.
3. Načíst aktivní `robotaxi_advisory_rules` (včetně junction tabulek) a vyhodnotit podmínky proti každému načtenému počasí. **Weather-based pravidlo matchuje segment, pokud splní podmínky proti alespoň jednomu načtenému `(resolved_weather_region_id, week_start)` záznamu** (OR logika — např. `dense_fog_disruption` při mlze v libovolném dotčeném regionu nebo týdnu; region může být rodič přes geo-fallback).
4. Sloučit matching `advisory_item_id` a priority (union; u stejné položky nejvyšší `priority`).
5. Seřadit dle `robotaxi_advisory_rules.priority` a `robotaxi_advisory_items.sort_order`.
6. `DELETE FROM segment_robotaxi_advisories WHERE segment_id = :id` → `INSERT` matching položek včetně vypočtené nejvyšší `priority`.

#### Agregace na výlet

Pro detail výletu sloučit upozornění ze **všech** řádků `segment_robotaxi_advisories` (union podle `advisory_item_id`). U stejné položky z více segmentů vyhrává **nejvyšší** `priority`. Výsledek uložit do `trip_robotaxi_advisories`. Nastavit `trips.robotaxi_advisories_computed_at`.

Labely pro UI mapuje FE z i18n souborů podle `slug`, `severity` a `users.locale` (fallback na výchozí jazyk aplikace).

#### Kdy invalidovat cache

Přepočty probíhají v aplikační vrstvě (ne PostgreSQL triggerem) a používají stejný trip-level zámek jako zápis segmentu — viz [Concurrency a čerstvost cache](implementation-notes.md#concurrency-a-čerstvost-cache).
Při asynchronní invalidaci nastav `trips.robotaxi_advisories_computed_at = NULL` ve stejné transakci jako evidence/enqueue změny; timestamp se obnoví až po úspěšném jobu.

- INSERT / UPDATE / DELETE segmentu nebo libovolná změna `transit_details` → přepočet spolu s ostatními agregacemi na `trips`; zejména změna `segment_kind` nebo `transport_mode` může segment přidat do/odebrat z robotaxi cache
- změna `weather_records` pro oblasti dotčené výletem **nebo jejich rodiče v geo-řetězci** → přepočet cache robotaxi upozornění (lze počítat v jednom průchodu s cache balení a `temp_*`)
- změna aktivních `robotaxi_advisory_rules`, jejich podmínek, priority nebo junction tabulek → najít dotčené výlety a přepočítat
- deaktivace `robotaxi_advisory_items` → odstranit položku z dotčených cache přepočtem; katalogovou položku nemaž, dokud na ni cache odkazuje přes RESTRICT
- změna `places.weather_region_id` nebo geo polí místa použitého robotaxi segmentem → najít dotčené výlety podle [Změna místa a invalidace výletů](places.md#změna-místa-a-invalidace-výletů)
- chybějící `places.weather_region_id` u robotaxi segmentů → weather-based pravidla se přeskočí; cache může být prázdná

#### Co zůstává mimo DB

- Textové věty typu „Hustá mlha může omezit provoz robotaxi — měj záložní plán“ generuj v aplikaci z agregovaných položek — ne jako volný TEXT v DB.
- Přeložené labely pro slugy robotaxi upozornění — FE i18n soubory, ne PostgreSQL.

### Servisní oblasti providerů

`provider_service_areas` zachycuje orientační katalog měst, kde který provider robotaxi provozuje. Geografie je **administrativní** (ISO kódy na úrovni locality), nezávislá na `weather_regions`; výsledek není geometrickou garancí dostupnosti jízdy.

**Service area ≠ last-mile k POI.** Pokrytí města říká, že provider v lokalitě jezdí. Kam až robotaxi doveze k konkrétnímu místu a kolik zbývá pěšky řeší kurátorovaná pole na `places` (`robotaxi_access`, `robotaxi_access_place_id`, `robotaxi_approach_walk_meters`) — viz [Last-mile robotaxi](places.md#places). Obě vrstvy se doplňují; last-mile nenahrazuje matching na `provider_service_areas`.

#### Matching místa na servisní oblast

Pro `places` s vyplněnými `country_code`, `subdivision_code` a `locality`:

1. Exact match: `(country_code, subdivision_code, locality)` → `provider_service_areas`.
2. Chybí některé pole → matching přeskočit (soft varování nelze vyhodnotit).

Jde o přesnou shodu normalizovaných administrativních hodnot, nikoli aliasů nebo geometrie. Import/admin musí pro stejnou lokalitu používat jednotný zápis; odlišný název či diakritika se ve v1 automaticky nesjednocují.

#### Validace při plánování (soft varování)

Při uložení robotaxi segmentu aplikace pro `provider_id` a místa `start_place_id` / `end_place_id` (pokud mají geo pole) ověří existenci `provider_service_areas` se `status = 'public'`. Chybí orientační pokrytí → **soft varování** autorovi (ne blokace zápisu): např. „Waymo nemá tuto lokalitu ve veřejném katalogu — ověřte dostupnost a záložní dopravu.“ Ani nalezená shoda nesmí být prezentována jako potvrzení obsluhy konkrétního místa.

Tvrdé validační minimum pro provider, model, zóny a kapacitu je kanonicky v [`transit_details` — pravidla](segments.md#transit_details--pravidla). Nový nebo měněný robotaxi úsek smí použít jen aktivního providera a aktivní model; deaktivace katalogové položky nemění historické itineráře.

Doplňková kontrola (volitelně v aplikaci):
- letiště na trase + `serves_airport = false` → varování
- segment mimo provozní okno když `operates_24_7 = false` → varování (viz [Provozní okno](#provozní-okno))

#### Provozní okno

Při `operates_24_7 = false` vymezují provoz `daily_start_local` a `daily_end_local` v **lokálním čase oblasti**. Porovnání s časem segmentu se dělá po převodu `start_time` / `end_time` do časové zóny oblasti.

| Vztah hodnot | Význam | Segment je uvnitř okna, když |
|---|---|---|
| `daily_start_local < daily_end_local` | Běžné denní okno (`06:00`–`22:00`) | `start >= daily_start_local AND end <= daily_end_local` |
| `daily_start_local > daily_end_local` | Okno **přes půlnoc** (`06:00`–`02:00`) | čas leží v `[daily_start_local, 24:00)` **nebo** v `[00:00, daily_end_local]` |
| `daily_start_local = daily_end_local` | Nejednoznačné (nula hodin, nebo 24?) | Aplikace zápis odmítne — pro nepřetržitý provoz použij `operates_24_7 = true` |

Rovnost obou hodnot nevynucuje DB CHECK (šlo by doplnit, ale nese riziko u importovaných dat) — hlídá ji admin validace při zápisu.

#### Katalogový dotaz — města s robotaxi

```sql
SELECT DISTINCT country_code, subdivision_code, subdivision_name, locality
FROM provider_service_areas
WHERE status = 'public'
ORDER BY country_code, locality;
```

#### Katalogový dotaz — výlety s konkrétním providerem

```sql
SELECT DISTINCT t.*
FROM trips t
JOIN segments s ON s.trip_id = t.id
JOIN transit_details td ON td.segment_id = s.id
JOIN robotaxi_providers rp ON rp.id = td.provider_id
WHERE t.status = 'published'
  AND t.visibility = 'public'
  AND td.transport_mode = 'robotaxi'
  AND rp.slug = 'waymo';
```

**Backlog v2:** polygonová geofence přes PostGIS pro point-in-polygon ověření konkrétních pickup/dropoff bodů.

