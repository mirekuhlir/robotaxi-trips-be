# Electrical standards

[← Přehled schématu](README.md)

Referenční katalog elektrických standardů podle země (typy zásuvek IEC, napětí, frekvence). **Není** součástí weather packingu ani checklistu cestovních požadavků (pas / vízum).

Zobrazení typů zásuvek a nápověda „vezmi adaptér“ probíhá **automaticky na FE** (read-time) — bez trip cache a bez zápisu do `trip_*` tabulek. Pas a vízum přidává autor ručně — viz [Travel requirements](travel-requirements.md).

## Tabulky

### `plug_types`

Katalog IEC typů zásuvek / zástrček.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `slug` | VARCHAR, UNIQUE, NOT NULL | Stabilní klíč, např. `type_c`, `type_f` |
| `sort_order` | SMALLINT, NOT NULL | Pořadí v seznamu; výchozí `0`; `>= 0` |
| `is_active` | BOOLEAN | Výchozí `true` — skrytí bez mazání |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Seed data (IEC typy A–N):**

| slug | sort_order |
|---|---|
| `type_a` | 10 |
| `type_b` | 20 |
| `type_c` | 30 |
| `type_d` | 40 |
| `type_e` | 50 |
| `type_f` | 60 |
| `type_g` | 70 |
| `type_h` | 80 |
| `type_i` | 90 |
| `type_j` | 100 |
| `type_k` | 110 |
| `type_l` | 120 |
| `type_m` | 130 |
| `type_n` | 140 |

Labely a ikony mapuje FE z i18n podle `slug` + `users.locale`.

### `country_electrical_standards`

Elektrický standard země — jedna řádka na ISO zemi. Primární napětí a frekvence (v1; dual-voltage země mimo scope).

| Sloupec | Typ | Popis |
|---|---|---|
| `country_code` | CHAR(2), PK | ISO 3166-1 alpha-2 (`CZ`, `US`, …) |
| `voltage_v` | SMALLINT, NOT NULL | Nominální střídavé napětí ve voltech (např. `230`, `120`) |
| `frequency_hz` | SMALLINT, NOT NULL | Frekvence sítě v Hz (`50` nebo `60`) |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Seed data (příklad — produkční seed doplň dle potřeby):**

| country_code | voltage_v | frequency_hz |
|---|---|---|
| `CZ` | 230 | 50 |
| `DE` | 230 | 50 |
| `AT` | 230 | 50 |
| `US` | 120 | 60 |
| `GB` | 230 | 50 |
| `JP` | 100 | 50 |

### `country_plug_types`

M:N — které typy zásuvek země používá / akceptuje (země často má více typů).

| Sloupec | Typ | Popis |
|---|---|---|
| `country_code` | CHAR(2), PK, FK → country_electrical_standards | ON DELETE CASCADE |
| `plug_type_id` | UUID, PK, FK → plug_types | ON DELETE RESTRICT |

**Seed data (příklad odpovídající tabulce výše):**

| country_code | plug slugy |
|---|---|
| `CZ` | `type_c`, `type_e` |
| `DE` | `type_c`, `type_f` |
| `AT` | `type_c`, `type_f` |
| `US` | `type_a`, `type_b` |
| `GB` | `type_g` |
| `JP` | `type_a`, `type_b` |

Zemi bez řádku v `country_electrical_standards` FE při lookupu přeskočí (neznámý standard).

---

## Poznámky k implementaci

### Automatické zobrazení na FE

BE drží jen katalog. FE na detailu výletu:

1. Sebrat distinct `places.country_code` z `start_place_id` / neprázdných `end_place_id` segmentů; ignorovat `NULL`.
2. Načíst `country_electrical_standards` + `country_plug_types` (+ `plug_types`) pro tyto kódy.
3. Zobrazit typy zásuvek, napětí a frekvenci destinací (labely z i18n).
4. Načíst `users.home_country_code` prohlížejícího / autora (dle produktu). Pokud je `NULL` nebo domácí země není v katalogu → jen destinace, bez nápovědy adaptéru.
5. Jinak: domácí typy \(H\); pro každou destinaci s katalogem typy \(D_i\). Pokud \(H \nsubseteq D_i\) u alespoň jedné destinace → zobrazit nápovědu „vezmi adaptér“.

**Příklady:** home `CZ` ({C, E}) → `US` ({A, B}) → adaptér ano. Home `CZ` → `DE` ({C, F}) → E∉D → adaptér ano. Home `CZ` → jen místa v `CZ` → ne.

Výsledek se **neukládá** do DB (žádné `trip_*` cache ani položka v `travel_requirement_items`).

### Oddělení od cestovních požadavků

| UI blok | Zdroj | Kdo rozhoduje |
|---|---|---|
| Zásuvky / adaptér | Tento katalog + geo výletu + `home_country_code` | FE automaticky |
| Dokumenty a formality (pas, vízum, …) | `trip_travel_requirements` | Autor ručně |

### Co zůstává mimo v1

| Funkce | Důvod |
|---|---|
| Dual-voltage země / více nominálních napětí | Potřebuje rozšíření schématu |
| Voltage converter jako samostatná nápověda | Málokdy nutné u moderních nabíječek; FE může doplnit později |
| EV / DC nabíjení | Kategorie místa `charging_fueling_hub` — jiná doména |

### Co zůstává mimo DB

- Přeložené labely typů zásuvek a text nápovědy adaptéru — FE i18n.
- Textové věty typu „V USA potřebujete adaptér na typ A/B“ — generuj na FE z katalogu.
