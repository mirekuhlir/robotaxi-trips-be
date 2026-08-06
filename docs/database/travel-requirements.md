# Travel requirements

[← Přehled schématu](README.md)

## Tabulky

### `travel_requirement_items`

Katalog cestovních požadavků (dokumenty, formality) — oddělený od `clothing_items` i od [elektrických standardů](electrical-standards.md) (zásuvky / adaptér řeší FE automaticky).

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `slug` | VARCHAR, UNIQUE, NOT NULL | Stabilní klíč, např. `passport`, `visa`, `national_id`, `travel_insurance` |
| `sort_order` | SMALLINT, NOT NULL | Pořadí v seznamu; výchozí `0`; `>= 0` |
| `is_active` | BOOLEAN | Výchozí `true` — skrytí bez mazání |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Seed data:**

| slug | sort_order |
|---|---|
| `passport` | 10 |
| `visa` | 20 |
| `national_id` | 30 |
| `travel_insurance` | 40 |

### `travel_requirement_rules`

Pravidla mapující geografii výletu na cestovní požadavek. Tabulka je ve schématu pro **budoucí** automatická geo pravidla. **V1 nemá seed pravidla** — pas, vízum i ostatní položky přidává autor ručně.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `travel_requirement_item_id` | UUID, FK → travel_requirement_items | ON DELETE CASCADE |
| `priority` | SMALLINT | Vyšší = důležitější v UI (výchozí `0`) |
| `min_distinct_countries_gte` | SMALLINT, nullable | Match, když výlet protíná ≥ N různých `places.country_code` mezi všemi `start_place_id` / `end_place_id` segmentů |
| `is_active` | BOOLEAN | Výchozí `true` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Validace pravidla (aplikace):** musí existovat alespoň jedna aktivní podmínka — jinak match-all riziko: alespoň jeden skalární sloupec NOT NULL.

**Seed v1:** žádné řádky.

### `trip_travel_requirements`

Agregovaná cache cestovních požadavků na úrovni výletu (0..N položek). Pouze trip-level — žádná per-segment tabulka.

| Sloupec | Typ | Popis |
|---|---|---|
| `trip_id` | UUID, PK, FK → trips | ON DELETE CASCADE |
| `travel_requirement_item_id` | UUID, PK, FK → travel_requirement_items | ON DELETE RESTRICT |
| `priority` | SMALLINT, NOT NULL | Efektivní priorita položky: maximum přes zdroje v `trip_travel_requirement_sources` |

### `trip_travel_requirement_sources`

Sledování, proč byla položka na úrovni výletu doporučena.

| Sloupec | Typ | Popis |
|---|---|---|
| `trip_id` | UUID, PK | Součást složeného FK `(trip_id, travel_requirement_item_id)` → `trip_travel_requirements(trip_id, travel_requirement_item_id)` ON DELETE CASCADE |
| `travel_requirement_item_id` | UUID, PK | Součást složeného FK `(trip_id, travel_requirement_item_id)` → `trip_travel_requirements(trip_id, travel_requirement_item_id)` ON DELETE CASCADE |
| `source` | travel_requirement_source, PK | |
| `priority` | SMALLINT, NOT NULL | Priorita daného zdroje; `trip_travel_requirements.priority` je maximum přes všechny zdroje položky |


---

## Poznámky k implementaci

### Cestovní požadavky

Cestovní formality (pas, vízum, …) jsou **oddělená doména** od weather packingu i od elektrických standardů. UI zobrazuje blok „Dokumenty a formality“ zvlášť od „Zásuvky / adaptér“ (viz [Electrical standards](electrical-standards.md)).

**V1:** autor výletu přidává a odebírá položky **ručně**. Žádný automatický přepočet z geografie. Cache je v `trip_travel_requirements` + `trip_travel_requirement_sources` se zdrojem `manual`. UI labely mapuje FE z i18n klíčů odvozených od `travel_requirement_items.slug`.

#### Cache tabulky

| Tabulka | Úroveň | Obsah |
|---|---|---|
| `trip_travel_requirements` | výlet | Seznam položek s efektivní `priority` |
| `trip_travel_requirement_sources` | výlet × položka × zdroj | Proč doporučeno: v1 vždy `manual`; `trip_geography` až s aktivními geo pravidly |
| `trips.travel_requirements_computed_at` | výlet | Čas poslední změny cache (ruční add/remove; později i geo přepočet) |

**Příklad dotazu — cestovní požadavky pro detail výletu:**

```sql
SELECT tri.slug, ttr.priority, ttr.travel_requirement_item_id
FROM trip_travel_requirements ttr
JOIN travel_requirement_items tri ON tri.id = ttr.travel_requirement_item_id
WHERE ttr.trip_id = :trip_id
ORDER BY ttr.priority DESC, tri.sort_order;
```

#### Ruční přidání / odebrání (v1)

1. Autor vybere aktivní položku z `travel_requirement_items` (nebo ji odebere).
2. `INSERT` / `DELETE` v `trip_travel_requirements` a odpovídající řádek ve `trip_travel_requirement_sources` se `source = manual` a zvolenou `priority`.
3. Nastavit `trips.travel_requirements_computed_at`.

Pas, vízum, občanka i pojištění — vše stejný ruční tok. FE může nabídnout nápovědu (např. „zvažte pas“), ale **zápis jen po akci autora**.

#### Geo přepočet (backlog — až budou aktivní pravidla)

Když v budoucnu přibudou aktivní `travel_requirement_rules`:

1. Sebrat místa z segmentů → distinct `places.country_code`.
2. Vyhodnotit aktivní pravidla (např. `min_distinct_countries_gte`).
3. Zapsat matching položky jako `source = trip_geography`; **zachovat** ruční položky (`manual`).
4. Přepočítat `trip_travel_requirements.priority = MAX(...)` přes zdroje; nastavit `travel_requirements_computed_at`.

#### Kdy aktualizovat cache

- ruční přidání/odebrání položky autorem (`source = manual`) — v1 jediný trigger
- (backlog) změna segmentů / `places.country_code` → geo přepočet spolu s ostatními agregacemi na `trips`

#### Co zůstává mimo v1 (backlog)

| Funkce | Důvod |
|---|---|
| Automatická geo pravidla (seed / aktivní `travel_requirement_rules`) | Produktově v1 jen ruční výběr autora |
| Vízum podle národnosti | Závisí na `users.home_country_code` + externí pravidla/API |
| Pas při cestě do cizí země vs. domácí | Totéž — `home_country_code` už je ve schématu, auto pravidlo ne |

`users.home_country_code` — viz [Users & trips](users-and-trips.md). Pro zásuvky ho FE používá hned — viz [Electrical standards](electrical-standards.md).

#### Co zůstává mimo DB

- Textové věty typu „Potřebujete vízum do Japonska“ generuj v aplikaci z agregovaných položek a kontextu uživatele — ne jako volný TEXT v DB.
- Přeložené labely pro slugy cestovních požadavků — FE i18n soubory, ne PostgreSQL.
