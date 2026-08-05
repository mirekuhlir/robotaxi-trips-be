# Travel times

[← Přehled schématu](README.md)

Doména **dojezdu mezi dvěma místy (`places`)** pro katalogový filtr „z místa X kam za ≤ N hodin“. Oddělená od délky programu výletu (`trips.total_duration_minutes`) i od `weather_regions` / `provider_service_areas`. Žádná tabulka hubů — origin i destinace jsou přímo `places.id`.

## Tabulky

### `travel_time_estimates`

Matice odhadů dojezdu place → place per `transport_mode`. Zdroj pravdy pro katalogový filtr dojezdu; naplňuje ji aplikační job / on-demand routing / admin (routing API, ruční seed, hrubý odhad) — ne PostgreSQL trigger.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `origin_place_id` | UUID, FK → places | ON DELETE CASCADE |
| `destination_place_id` | UUID, FK → places | ON DELETE CASCADE |
| `transport_mode` | transport_mode, NOT NULL | Režim dopravy odhadu |
| `duration_minutes` | INTEGER, NOT NULL | Odhadovaná doba cesty v minutách; `> 0` |
| `distance_meters` | INTEGER, nullable | Volitelná délka trasy v metrech; pokud vyplněno, `>= 0` |
| `source` | VARCHAR, NOT NULL | Slug zdroje odhadu (např. `routing_api`, `manual`, `estimated`) — volný text jako `places.external_source`, ne enum |
| `computed_at` | TIMESTAMPTZ, NOT NULL | Čas výpočtu / poslední aktualizace odhadu; výchozí `now()` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Omezení:** `UNIQUE (origin_place_id, destination_place_id, transport_mode)`.

**CHECK:** `origin_place_id <> destination_place_id` — stejné místo se do matice neukládá (`duration_minutes = 0` řeší aplikace při filtru origin = destinace).

Matice je **řídká**: typicky populární originy × destinace publikovaných výletů (ne plný N×N graf všech POI).

Příklad: place „Praha (centrum)“ → place „Benátky / hotel“, `own_car`, `duration_minutes = 900` (15 h).


---

## Poznámky k implementaci

### Vztah k místům a výletům

| Entita | Role |
|---|---|
| `trips.destination_place_id` | Cache destinace výletu (`places.id`) |
| `trips.outbound_transport_mode` | Cache režimu „jak se tam dostanu“ z itineráře |
| `travel_time_estimates` | Lookup `(origin_place, destination_place, mode) → duration_minutes` |

`trips.total_duration_minutes` zůstává **délkou programu** (aktivity + doprava v itineráři bez ubytování). Filtr dojezdu ji nepoužívá.

### Long-haul vs. short-hop režimy

Pro odvození `outbound_transport_mode` a smysluplné řádky matice:

| Skupina | `transport_mode` |
|---|---|
| **Long-haul** (dojezd mezi místy) | `plane`, `train`, `bus`, `own_car`, `car_rental`, `rideshare`, `boat`, `helicopter`, `motorcycle`, `other` |
| **Short-hop** (vyloučené z outbound) | `walk`, `tram`, `metro`, `bicycle`, `bike_share`, `scooter`, `cable_car` |
| **Lokální / speciální** | `robotaxi` — typicky short-hop; do long-haul množiny **nepatří**, pokud není jediný transit k destinaci (viz [Cache destinace a outbound režimu](#cache-destinace-a-outbound-režimu-na-trips)) |

Matice v1 typicky drží odhady jen pro long-haul režimy; short-hop řádky nejsou potřeba pro katalogový filtr.

### Cache destinace a outbound režimu na `trips`

Sloupce `destination_place_id` a `outbound_transport_mode` jsou read-only cache. Autor je nevyplňuje ručně; aplikace je přepočítá při změně segmentů nebo při smazání / změně míst použité v itineráři. DB CHECK vynucuje párovost: **obě NOT NULL, nebo obě NULL** (stejný pattern jako `temperature_source` ↔ `feels_like_*`).

#### Pravidlo destinace (`destination_place_id`)

Deterministicky, v tomto pořadí (`ORDER BY start_time, id` na segmentech):

1. `start_place_id` **prvního** `accommodation` segmentu.
2. Jinak `start_place_id` **první** `activity`.
3. Jinak `end_place_id` **prvního long-haul** transit segmentu.
4. Jinak `NULL`.

#### Pravidlo outbound režimu (`outbound_transport_mode`)

Vyžaduje nejdřív vyřešený `destination_place_id`. Pokud je destinace `NULL`, nastav i režim na `NULL`.

1. Kandidáti: segmenty `segment_kind = transit` s `transit_details.transport_mode` v long-haul množině.
2. Preferovat kandidáty, jejichž `end_place_id = destination_place_id` a které chronologicky končí **před nebo na** začátku první aktivity/ubytování v destinaci (první segment s `start_place_id = destination_place_id` typu accommodation/activity).
3. Z preferovaných (nebo ze všech long-haul, pokud žádný nekončí přesně v destinaci) vybrat režim segmentu s **nejdelší** délkou `(end_time - start_time)`; při shodě vyhrává dřívější `start_time`, pak menší `id`.
4. **Výjimka `robotaxi`:** pokud neexistuje žádný long-haul kandidát, ale existuje transit s `transport_mode = robotaxi`, jehož `end_place_id = destination_place_id`, použij `robotaxi` (jediný transit k destinaci — typicky lokální výlet).
5. Jinak `NULL`.

Po výpočtu: pokud je vyplněný jen jeden z páru `(destination_place_id, outbound_transport_mode)`, nastav **oba** na `NULL` (nekonzistentní částečná cache se neukládá).

FK `destination_place_id` má `ON DELETE SET NULL`. Smazání místa destinace vynuluje pár (aplikace při SET NULL nastaví i `outbound_transport_mode = NULL`, nebo přepočítá celý výlet).

#### Kdy invalidovat cache

Přepočty v aplikační vrstvě (ne DB triggerem), typicky ve stejné transakci jako ostatní agregace na `trips`:

- INSERT / UPDATE / DELETE segmentů nebo `transit_details` výletu
- smazání místa použitých v segmentech (RESTRICT na `segments` obvykle blokuje dřív) — po přepojení segmentů přepočítat

### Naplnění matice

Mimo DB schéma:

1. Job nebo on-demand routing upsertuje řádky `(origin_place_id, destination_place_id, transport_mode)` s `duration_minutes`, volitelně `distance_meters`, `source`, `computed_at`.
2. Re-compute při změně routingu / ruční korekci — stejný unique klíč.
3. Chybějící řádek ≠ chyba schématu; filtr dojezdu takový výlet prostě nevybere.
4. Preferuj řídkou matici: populární originy (např. města jako place) × `destination_place_id` publikovaných výletů.

### Katalogový filtr dojezdu

FE parametry: `origin_place_id` (vyhledání / výběr místa, např. Praha) + `max_minutes` (např. `15 * 60`).

**Stejné místo:** pokud `trips.destination_place_id = :origin_place_id`, považuj dojezd za `0` minut (bez řádku v matici) — výlet projde filtrem při libovolném `max_minutes >= 0`.

**Jiná destinace:**

```sql
SELECT t.*
FROM trips t
JOIN travel_time_estimates e
  ON e.destination_place_id = t.destination_place_id
 AND e.transport_mode = t.outbound_transport_mode
WHERE t.status = 'published'
  AND t.visibility = 'public'
  AND t.destination_place_id IS NOT NULL
  AND t.outbound_transport_mode IS NOT NULL
  AND e.origin_place_id = :origin_place_id
  AND e.duration_minutes <= :max_minutes;
```

Ekvivalent včetně stejného místa:

```sql
SELECT t.*
FROM trips t
LEFT JOIN travel_time_estimates e
  ON e.destination_place_id = t.destination_place_id
 AND e.origin_place_id = :origin_place_id
 AND e.transport_mode = t.outbound_transport_mode
WHERE t.status = 'published'
  AND t.visibility = 'public'
  AND t.destination_place_id IS NOT NULL
  AND t.outbound_transport_mode IS NOT NULL
  AND (
    t.destination_place_id = :origin_place_id
    OR (e.id IS NOT NULL AND e.duration_minutes <= :max_minutes)
  );
```

Chybí-li řádek matice pro `(origin, dest, mode)` a destinace ≠ origin → výlet ve **filtrovaném** katalogu dojezdu neprojde (stejná filozofie jako `feels_like_*` / `max_difficulty`). Bez aktivního filtru dojezdu zůstává v běžném katalogu podle ostatních kritérií.

**„Okruh“ na mapě:** množina destinací (místa / výlety), pro které existuje odhad z `:origin_place_id` s `duration_minutes <= :max_minutes` (případně filtrované režimy přítomné u výsledků). Není to geometrický kruh ani isochrone polygon — PostGIS isochrone zůstává mimo v1.

### Co zůstává mimo DB

- Volání routing API (OSRM, Google Directions, …) a scheduling jobů
- Přeložené labely režimů — FE i18n (`users.locale`)
- Profilové `users.home_place_id` — backlog; v1 stačí picker / search místa ve filtru
- Isochrone polygony
- Samostatná tabulka městských hubů — origin i destinace jsou `places`
