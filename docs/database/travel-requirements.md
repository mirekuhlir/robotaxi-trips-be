# Travel requirements

[← Přehled schématu](README.md)

## Tabulky

### `travel_requirement_items`

Katalog cestovních požadavků (dokumenty, formality) — oddělený od `clothing_items`.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `slug` | VARCHAR, UNIQUE, NOT NULL | Stabilní klíč, např. `passport`, `visa`, `national_id`, `travel_insurance` |
| `sort_order` | SMALLINT | Výchozí pořadí v seznamu; `>= 0` |
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

Pravidla mapující geografii výletu na cestovní požadavek. Vyhodnocují se na úrovni **celého výletu** (ne per segment, ne z počasí).

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

**Seed pravidlo v1:**

| Položka | Podmínka |
|---|---|
| `passport` | `min_distinct_countries_gte = 2` |

`visa`, `national_id`, `travel_insurance` — položky v katalogu, **bez automatického pravidla v1** (ruční doplnění autorem nebo backlog v2).

### `trip_travel_requirements`

Agregovaná cache cestovních požadavků na úrovni výletu (0..N položek). Pouze trip-level — žádná per-segment tabulka.

| Sloupec | Typ | Popis |
|---|---|---|
| `trip_id` | UUID, PK, FK → trips | ON DELETE CASCADE |
| `travel_requirement_item_id` | UUID, PK, FK → travel_requirement_items | ON DELETE RESTRICT |
| `priority` | SMALLINT, NOT NULL | Efektivní priorita položky: maximum z aktivního geo pravidla a ruční hodnoty, pokud má položka oba zdroje |

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

Cestovní formality (pas, vízum, …) jsou **oddělená doména** od weather packingu. Cache je pouze na úrovni výletu v `trip_travel_requirements` + `trip_travel_requirement_sources` — UI labely mapuje FE z i18n klíčů odvozených od `travel_requirement_items.slug`.

#### Cache tabulky

| Tabulka | Úroveň | Obsah |
|---|---|---|
| `trip_travel_requirements` | výlet | Seznam doporučených položek s efektivní `priority` |
| `trip_travel_requirement_sources` | výlet × položka × zdroj | Proč doporučeno a s jakou zdrojovou prioritou: `trip_geography` a/nebo `manual` |
| `trips.travel_requirements_computed_at` | výlet | Čas posledního přepočtu |

**Příklad dotazu — cestovní požadavky pro detail výletu:**

```sql
SELECT tri.slug, ttr.priority, ttr.travel_requirement_item_id
FROM trip_travel_requirements ttr
JOIN travel_requirement_items tri ON tri.id = ttr.travel_requirement_item_id
WHERE ttr.trip_id = :trip_id
ORDER BY ttr.priority DESC, tri.sort_order;
```

#### Výpočet na výlet (v1)

1. Sebrat všechna místa z segmentů: `start_place_id` + neprázdné `end_place_id`.
2. Načíst `places.country_code`; ignorovat `NULL`.
3. `distinct_countries = COUNT(DISTINCT country_code)`.
4. Vyhodnotit aktivní `travel_requirement_rules` proti `distinct_countries` (např. `passport` když `distinct_countries >= 2`).
5. Sloučit matching položky (union; u stejné položky nejvyšší `priority` z geo pravidel) a zapsat je jako zdroje `trip_geography` do `trip_travel_requirement_sources.priority`.
6. **Zachovat ruční položky** se zdrojem `manual` — nesmazat při přepočtu geo pravidel; ruční priorita zůstává uložená v `trip_travel_requirement_sources.priority`.
7. `DELETE` geo zdrojů (`source = trip_geography`) → `INSERT` matching geo zdrojů; pro každou položku přepočítat `trip_travel_requirements.priority = MAX(trip_travel_requirement_sources.priority)`. Pokud po odebrání geo zdroje zůstává `manual`, ponechat položku s ruční prioritou; pokud nezůstane žádný zdroj, smazat i řádek z `trip_travel_requirements`. Nastavit `trips.travel_requirements_computed_at`.

Chybí vyplněné `country_code` u všech míst → geo pravidla se přeskočí (stejně jako chybějící `weather_region_id` u balení).

**Příklad:** výlet Praha (`CZ`) → Karlštejn (`CZ`) → Vídeň (`AT`) má `distinct_countries = 2` → automaticky doporučí `passport`.

#### Kdy invalidovat cache

Přepočty probíhají v aplikační vrstvě (ne PostgreSQL triggerem).

- změna segmentů výletu → přepočet spolu s ostatními agregacemi na `trips`
- UPDATE `places.country_code` u míst dotčených výletů → najít dotčené výlety a přepočítat
- ruční přidání/odebrání položky autorem (`source = manual`)

#### Co zůstává mimo v1 (backlog v2)

| Funkce | Důvod |
|---|---|
| Vízum podle národnosti | Závisí na `users.home_country_code` + externí pravidla/API |
| Pas při cestě do jedné cizí země | Potřebuje domovskou zemi uživatele |
| Automatické cestovní pojištění | Business pravidla mimo geografii |

V1 automaticky doporučí **pas jen při multi-country výletu**; ostatní položky (`visa`, `national_id`, `travel_insurance`) lze přidat ručně autorem nebo doplnit pravidly v budoucnu.

#### Co zůstává mimo DB

- Textové věty typu „Potřebujete vízum do Japonska“ generuj v aplikaci z agregovaných položek a kontextu uživatele — ne jako volný TEXT v DB.
- Přeložené labely pro slugy cestovních požadavků — FE i18n soubory, ne PostgreSQL.

---
