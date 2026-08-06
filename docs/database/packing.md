# Packing

[← Přehled schématu](README.md)

## Tabulky

### `clothing_items`

Katalog doporučených položek oblečení a vybavení.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `slug` | VARCHAR, UNIQUE, NOT NULL | Stabilní klíč, např. `light_jacket`, `umbrella`, `sunscreen` |
| `category` | clothing_category | Skupina pro UI |
| `sort_order` | SMALLINT, NOT NULL | Pořadí v seznamu; výchozí `0`; `>= 0` |
| `is_active` | BOOLEAN | Výchozí `true` — skrytí bez mazání |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Seed data (příklad):**

| slug | category |
|---|---|
| `light_jacket` | `outerwear` |
| `warm_jacket` | `outerwear` |
| `rain_jacket` | `outerwear` |
| `layers` | `layering` |
| `hat` | `accessories` |
| `gloves` | `accessories` |
| `umbrella` | `accessories` |
| `sunscreen` | `protection` |
| `sunglasses` | `accessories` |
| `comfortable_shoes` | `footwear` |
| `hiking_boots` | `footwear` |
| `waterproof_shoes` | `footwear` |

### `clothing_rules`

Pravidla mapující podmínky (počasí + kontext segmentu) na položku oblečení. Všechny podmínky jsou relační — skalární sloupce na tomto řádku nebo řádky v junction tabulkách. Vyplněné podmínky se kombinují logikou **AND**; u junction tabulek platí logika **IN** (alespoň jedna hodnota musí sedět).

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `clothing_item_id` | UUID, FK → clothing_items | ON DELETE CASCADE |
| `priority` | SMALLINT | Vyšší = důležitější v UI (výchozí `0`) |
| `temp_avg_c_lte` | NUMERIC(4, 1), nullable | Match proti `weather_records.temp_avg_c` (vzduchová) |
| `temp_avg_c_gte` | NUMERIC(4, 1), nullable | Match proti `weather_records.temp_avg_c` (vzduchová) |
| `temp_min_c_lte` | NUMERIC(4, 1), nullable | Match proti `weather_records.temp_min_c`; při `NULL` v záznamu fallback na `temp_avg_c` |
| `temp_min_c_gte` | NUMERIC(4, 1), nullable | Match proti `weather_records.temp_min_c`; při `NULL` v záznamu fallback na `temp_avg_c` |
| `temp_max_c_lte` | NUMERIC(4, 1), nullable | Match proti `weather_records.temp_max_c`; při `NULL` v záznamu fallback na `temp_avg_c` |
| `temp_max_c_gte` | NUMERIC(4, 1), nullable | Match proti `weather_records.temp_max_c`; při `NULL` v záznamu fallback na `temp_avg_c` |
| `feels_like_avg_c_lte` | NUMERIC(4, 1), nullable | Match proti `weather_records.feels_like_avg_c`; při `NULL` v záznamu fallback na vzduchovou `temp_avg_c` |
| `feels_like_avg_c_gte` | NUMERIC(4, 1), nullable | Match proti `weather_records.feels_like_avg_c`; při `NULL` v záznamu fallback na vzduchovou `temp_avg_c` |
| `feels_like_min_c_lte` | NUMERIC(4, 1), nullable | Match proti efektivní pocitové min (viz lookup krok 4); fallback řetězec: `feels_like_min_c` → `feels_like_avg_c` → `temp_min_c` → `temp_avg_c` |
| `feels_like_min_c_gte` | NUMERIC(4, 1), nullable | Match proti efektivní pocitové min |
| `feels_like_max_c_lte` | NUMERIC(4, 1), nullable | Match proti efektivní pocitové max; fallback: `feels_like_max_c` → `feels_like_avg_c` → `temp_max_c` → `temp_avg_c` |
| `feels_like_max_c_gte` | NUMERIC(4, 1), nullable | Match proti efektivní pocitové max |
| `rainy_days_gte` | SMALLINT, nullable | Match proti `weather_records.rainy_days` |
| `min_duration_minutes_gte` | INTEGER, nullable | Match proti délce segmentu `(end_time - start_time)` |
| `non_accommodation_activity` | BOOLEAN, nullable | `true` = jen segmenty `segment_kind = activity` mimo kategorii `accommodation` na `start_place_id` (aktivity v hotelu — spa, snídaně — tedy pravidlo nespustí); `NULL` = nevyhodnocovat. Hodnota `false` je zakázaná DB CHECK constraintem — byla by tichý třetí stav bez sémantiky |
| `is_active` | BOOLEAN | Výchozí `true` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Teplotní podmínky — priorita pocitové vs. vzduchové:** Pokud má pravidlo **alespoň jeden** vyplněný sloupec `feels_like_*_gte` / `feels_like_*_lte`, vyhodnocují se **jen** pocitové prahy (vzduchové `temp_*` na stejném pravidle se ignorují). Jinak se vyhodnocují vzduchové `temp_*`. Seed pravidla v1 používají vzduchovou teplotu; pocitové prahy jsou připravené pro pravidla citlivá na dusno / vlezlou zimu.

**Validace pravidla (aplikace):** musí existovat alespoň jedna aktivní podmínka — jinak match-all riziko:
- alespoň jeden skalární sloupec NOT NULL, nebo
- `non_accommodation_activity = true`, nebo
- ≥1 řádek v libovolné junction tabulce níže

**Příklad seed pravidel:**

| Položka | `clothing_rules` | Junction tabulky |
|---|---|---|
| `light_jacket` — chladno | `temp_max_c_lte = 15` | — |
| `rain_jacket` / `umbrella` — déšť | — | 5 řádků v `clothing_rule_precipitation_intensities` (`drizzle` … `storm`) |
| `hiking_boots` — náročná venkovní aktivita | `non_accommodation_activity = true` | `hard`, `extreme` v `clothing_rule_difficulties` |
| `sunscreen` — slunce | `temp_avg_c_gte = 18` | `clear`, `mostly_sunny` v `clothing_rule_sky_conditions` |

### `clothing_rule_precipitation_intensities`

Množinová podmínka na intenzitu srážek (IN logika — alespoň jedna hodnota musí sedět s `weather_records.precipitation_intensity`).

| Sloupec | Typ | Popis |
|---|---|---|
| `clothing_rule_id` | UUID, PK, FK → clothing_rules | ON DELETE CASCADE |
| `precipitation_intensity` | precipitation_intensity, PK | |

### `clothing_rule_wind_forces`

Množinová podmínka na sílu větru (IN logika).

| Sloupec | Typ | Popis |
|---|---|---|
| `clothing_rule_id` | UUID, PK, FK → clothing_rules | ON DELETE CASCADE |
| `wind_force` | wind_force, PK | |

### `clothing_rule_fog_conditions`

Množinová podmínka na mlhu (IN logika).

| Sloupec | Typ | Popis |
|---|---|---|
| `clothing_rule_id` | UUID, PK, FK → clothing_rules | ON DELETE CASCADE |
| `fog_condition` | fog_condition, PK | |

### `clothing_rule_sky_conditions`

Množinová podmínka na oblačnost (IN logika).

| Sloupec | Typ | Popis |
|---|---|---|
| `clothing_rule_id` | UUID, PK, FK → clothing_rules | ON DELETE CASCADE |
| `sky_condition` | sky_condition, PK | |

### `clothing_rule_difficulties`

Množinová podmínka na náročnost segmentu (IN logika — match proti `segments.difficulty`).

| Sloupec | Typ | Popis |
|---|---|---|
| `clothing_rule_id` | UUID, PK, FK → clothing_rules | ON DELETE CASCADE |
| `segment_difficulty` | segment_difficulty, PK | |


### `segment_packing_items`

Cache doporučeného oblečení per segment (0..N položek). Přepočet v aplikaci — viz [Doporučené oblečení](#doporučené-oblečení).

| Sloupec | Typ | Popis |
|---|---|---|
| `segment_id` | UUID, PK, FK → segments | ON DELETE CASCADE |
| `clothing_item_id` | UUID, PK, FK → clothing_items | ON DELETE RESTRICT |
| `priority` | SMALLINT, NOT NULL | Nejvyšší priorita matching `clothing_rules` pro daný segment a položku |

### `segment_packing_item_sources`

Sledování, proč byla položka na segmentu doporučena (weather vs. segment kontext). Stejný tvar jako `trip_packing_item_sources` — bez sloupce `priority`, protože efektivní priorita je v `segment_packing_items`.

Díky této tabulce je trip-level cache **plně odvoditelná ze segmentové**: `trip_packing_item_sources` je union těchto řádků přes segmenty výletu, takže agregace nemusí znovu vyhodnocovat `clothing_rules`.

| Sloupec | Typ | Popis |
|---|---|---|
| `segment_id` | UUID, PK | Součást složeného FK `(segment_id, clothing_item_id)` → `segment_packing_items(segment_id, clothing_item_id)` ON DELETE CASCADE |
| `clothing_item_id` | UUID, PK | Součást složeného FK `(segment_id, clothing_item_id)` → `segment_packing_items(segment_id, clothing_item_id)` ON DELETE CASCADE |
| `source` | packing_source, PK | |


### `trip_packing_items`

Agregovaná cache doporučeného oblečení na úrovni výletu (0..N položek). Při sloučení ze segmentů platí u stejné položky **nejvyšší** `priority`.

| Sloupec | Typ | Popis |
|---|---|---|
| `trip_id` | UUID, PK, FK → trips | ON DELETE CASCADE |
| `clothing_item_id` | UUID, PK, FK → clothing_items | ON DELETE RESTRICT |
| `priority` | SMALLINT, NOT NULL | Priorita z matching `clothing_rules` |

### `trip_packing_item_sources`

Sledování, proč byla položka na úrovni výletu doporučena (weather vs. segment kontext). Obsah je **union `segment_packing_item_sources`** přes segmenty výletu.

Na rozdíl od `trip_travel_requirement_sources` tabulka **nemá sloupec `priority`** — záměrná asymetrie: oba zdroje balení (`weather`, `segment`) se při přepočtu kompletně regenerují z `clothing_rules`, takže per-zdroj prioritu není třeba perzistovat; efektivní priorita položky je v `trip_packing_items.priority`. U cestovních požadavků je v1 jen `manual`; až přibudou geo pravidla, `manual` musí přežít geo přepočet i se svou prioritou.

| Sloupec | Typ | Popis |
|---|---|---|
| `trip_id` | UUID, PK | Součást složeného FK `(trip_id, clothing_item_id)` → `trip_packing_items(trip_id, clothing_item_id)` ON DELETE CASCADE |
| `clothing_item_id` | UUID, PK | Součást složeného FK `(trip_id, clothing_item_id)` → `trip_packing_items(trip_id, clothing_item_id)` ON DELETE CASCADE |
| `source` | packing_source, PK | |


---

## Poznámky k implementaci

### Doporučené oblečení

Doporučení se **neukládá na `weather_records` ani `places`**. Počasí zůstává týdenní a regionální; oblečení je odvozenina z počasí + kontextu segmentu. Cache je v relačních tabulkách `segment_packing_items` + `segment_packing_item_sources` (per segment) a `trip_packing_items` + `trip_packing_item_sources` (agregace na výlet) — UI labely i ikony mapuje FE z i18n klíčů odvozených od `clothing_items.slug` (případně `category`).

#### Cache tabulky

| Tabulka | Úroveň | Obsah |
|---|---|---|
| `segment_packing_items` | segment | Které `clothing_items` doporučit pro daný segment + nejvyšší `priority` z matching pravidel |
| `segment_packing_item_sources` | segment × položka | Proč doporučeno na segmentu: `weather` a/nebo `segment` |
| `trip_packing_items` | výlet | Agregovaný seznam s `priority` (union ze segmentů; u stejné položky nejvyšší priorita) |
| `trip_packing_item_sources` | výlet × položka | Proč doporučeno: union zdrojů ze `segment_packing_item_sources` |
| `trips.packing_computed_at` | výlet | Čas posledního přepočtu |

**Příklad dotazu — doporučení pro detail výletu:**

```sql
SELECT ci.slug, tpi.priority, tpi.clothing_item_id
FROM trip_packing_items tpi
JOIN clothing_items ci ON ci.id = tpi.clothing_item_id
WHERE tpi.trip_id = :trip_id
ORDER BY tpi.priority DESC, ci.sort_order;
```

**Příklad dotazu — doporučení per segment (timeline UI):**

```sql
SELECT ci.slug, spi.priority
FROM segment_packing_items spi
JOIN clothing_items ci ON ci.id = spi.clothing_item_id
WHERE spi.segment_id = :segment_id
ORDER BY spi.priority DESC, ci.sort_order;
```

#### Výpočet na segment

1. Načíst `weather_records` pro segment podle [Lookup počasí podle typu segmentu](weather-and-climate.md#lookup-počasí-podle-typu-segmentu) (kroky 1–4; krok 3 zahrnuje [geo-fallback](weather-and-climate.md#hierarchie-oblastí-a-geo-fallback) na rodičovskou oblast, když na oblasti místa záznam chybí; krok 4 řeší fallback vzduchové i pocitové teploty). Klima se pro balení nepoužívá.
2. Načíst aktivní `clothing_rules` (včetně junction tabulek) a vyhodnotit podmínky proti každému načtenému počasí a kontextu segmentu (`difficulty`, délka, `non_accommodation_activity`). U teploty: pokud rule má alespoň jeden `feels_like_*` práh, matchovat jen proti efektivní pocitové; jinak proti efektivní vzduchové (viz [clothing_rules](#clothing_rules)). **Weather-based pravidlo matchuje segment, pokud splní podmínky proti alespoň jednomu načtenému `(resolved_weather_region_id, week_start)` záznamu** (OR logika — např. deštník při dešti v libovolném dotčeném regionu nebo týdnu; region může být rodič přes geo-fallback). Segment-only podmínky (`clothing_rule_difficulties`, `non_accommodation_activity`, `min_duration_minutes_gte`) se vyhodnocují normálně i bez weather záznamu.
3. Sloučit matching `clothing_item_id` a priority (union; u stejné položky nejvyšší `priority`) a k položce zapamatovat zdroje matching pravidel (viz [Klasifikace zdroje](#klasifikace-zdroje-packing_source)).
4. Seřadit dle `clothing_rules.priority` a `clothing_items.sort_order`.
5. `DELETE FROM segment_packing_items WHERE segment_id = :id` (CASCADE smaže i `segment_packing_item_sources`) → `INSERT` matching položek včetně vypočtené nejvyšší `priority` → `INSERT` jejich zdrojů do `segment_packing_item_sources`.

#### Klasifikace zdroje (`packing_source`)

Zdroj se určuje podle podmínek matching pravidla. Pravidlo s alespoň jednou weather podmínkou (teplotní sloupce `temp_*` / `feels_like_*`, `rainy_days_gte`, junction tabulky `clothing_rule_precipitation_intensities` / `clothing_rule_wind_forces` / `clothing_rule_fog_conditions` / `clothing_rule_sky_conditions`) přispívá zdrojem `weather`; pravidlo s alespoň jednou segment podmínkou (`clothing_rule_difficulties`, `min_duration_minutes_gte`, `non_accommodation_activity`) přispívá zdrojem `segment`. **Smíšené pravidlo přispívá oběma zdroji.** Položka tak může mít řádek pro `weather` i `segment` — z jednoho smíšeného pravidla nebo z více různých pravidel.

Klasifikace probíhá **na úrovni segmentu** a zapisuje se do `segment_packing_item_sources`; trip-level tabulka je jen union.

#### Agregace na výlet

Pro detail výletu sloučit doporučení ze **všech** řádků `segment_packing_items` (union podle `clothing_item_id`). U stejné položky z více segmentů vyhrává **nejvyšší** `priority`. Výsledek uložit do `trip_packing_items`; zdroje do `trip_packing_item_sources` jako union `segment_packing_item_sources` přes segmenty výletu — agregace tedy nemusí znovu vyhodnocovat `clothing_rules`. Nastavit `trips.packing_computed_at`.

```sql
-- zdroje na úrovni výletu = union segmentových zdrojů
INSERT INTO trip_packing_item_sources (trip_id, clothing_item_id, source)
SELECT DISTINCT s.trip_id, spis.clothing_item_id, spis.source
FROM segment_packing_item_sources spis
JOIN segments s ON s.id = spis.segment_id
WHERE s.trip_id = :trip_id;
```

Labely pro UI mapuje FE z i18n souborů podle `slug` a `users.locale` (fallback na výchozí jazyk aplikace).

#### Kdy invalidovat cache

Přepočty probíhají v aplikační vrstvě (ne PostgreSQL triggerem).

- změna segmentů výletu → přepočet cache balení, cache robotaxi upozornění, `temp_min_c` / `temp_max_c`, `feels_like_min_c` / `feels_like_max_c`, `temperature_source` a ostatních agregací na `trips` (cestovní požadavky v1 jen ručně — ne při změně segmentů; zásuvky FE read-time)
- změna `weather_records` pro oblasti dotčené výletem **nebo jejich rodiče v geo-řetězci** → přepočet cache balení, cache robotaxi upozornění a `temp_*` / `feels_like_*`; aplikace musí najít dotčené výlety (segmenty s místy v dané oblasti nebo v potomcích, kteří na ni mohou spadnout × protínající lokální ISO týdny)
- sync `weather_climate_months` pro oblast (nebo rodiče) → přepočet **jen** teplotní cache a `temperature_source` (balení klima nepoužívá — viz [Fallback na klima](weather-and-climate.md#fallback-na-klima))
- chybějící `places.weather_region_id` u segmentů → weather-based pravidla se přeskočí; `temp_*` / `feels_like_*` cache může zůstat `NULL` → výlet mimo filtrovaný katalog dle teploty

#### Co zůstává mimo DB

- Textové věty typu „V pátek bude déšť, vezměte si bundu“ generuj v aplikaci z agregovaných položek — ne jako volný TEXT v DB.
- Přeložené labely pro katalogové slugy a enum hodnoty (oblečení, počasí, náročnost, kategorie míst, robotaxi upozornění) — FE i18n soubory, ne PostgreSQL.

