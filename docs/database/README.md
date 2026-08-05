# robotaxitrips.com — Databázové schéma (PostgreSQL)

> Produkčně připravené schéma pro plánování itinerářů a výletů s integrací autonomní dopravy (robotaxi).

Dokumentace je rozdělena podle domén. Diagramový zdroj: [`robotaxi-trips-database.dbml`](robotaxi-trips-database.dbml).

## Obsah

- [Users & trips](users-and-trips.md) — `users`, `trips`, členové, recenze výletů; status/visibility; Recenze
- [Places](places.md) — kategorie, místa, přijímané platby (`accepted_payments`), last-mile robotaxi (`robotaxi_access`), recenze míst; geografické dotazy
- [Segments](segments.md) — segmenty, galerie, `transit_details`; sémantika, cena, věk
- [Weather & climate](weather-and-climate.md) — oblasti, týdenní počasí, klima; teplota vzduchu/pocitová; teplota mořské vody (SST); geo-fallback hierarchie
- [Packing](packing.md) — oblečení, pravidla, cache balení
- [Travel requirements](travel-requirements.md) — cestovní formality
- [Travel times](travel-times.md) — matice dojezdů place→place, katalogový filtr „z místa X do N hodin“
- [Robotaxi](robotaxi.md) — provideři, vozidla, oblasti, advisories
- [Constraints & indexes](constraints-and-indexes.md) — CHECK, app validace, indexy
- [Implementation notes](implementation-notes.md) — agregace na `trips` + odkazy na doménové poznámky

---

## Architektonické principy

| Princip | Popis |
|---|---|
| **Lineární časová osa** | Itinerář se skládá ze segmentů s přesným `timestamptz` seřazených na timeline. Multimodální trasy (chůze → robotaxi → chůze) jsou sérií samostatných segmentů, ne stromem. Segmenty jednoho výletu se nesmí překrývat — platí i pro `accommodation`; vícedenní ubytování se modeluje jako check-in/check-out nebo noční úseky bez překryvu s aktivitami a dopravou. Každý segment má explicitní `segment_kind`. Rozdělení na „dny" řeší frontend. |
| **Explicitní typ segmentu** | `segment_kind` (`accommodation` / `activity` / `transit`) je primární zdroj pravdy; `end_place_id`, `transit_details` a kategorie místa musí být s ním konzistentní (validace v aplikaci). |
| **Počasí odděleně od balení** | Týdenní `weather_records` per region; doporučení oblečení přes relační `clothing_rules` (+ junction tabulky) a normalizovanou cache `segment_packing_items` / `trip_packing_items` (+ `*_sources`) — bez duplikace počasí na segmentu. Cestovní formality (pas, vízum, …) jsou **oddělená doména** — cache `trip_travel_requirements`, ne součást weather packingu. |
| **Klima měsíců odděleně od týdenního počasí** | Měsíční klimatické normály v `weather_climate_months` (typický měsíc v oblasti); týdenní prognóza/historie v `weather_records`. Skóre a label „ideální měsíc“ jsou odvozenina v aplikaci (read-time na detailu výletu) — bez cache na `trips` a bez duplikace klimatu na segmentu. Klima navíc slouží jako **fallback** pro teplotní cache katalogu (vzduchová, pocitová i SST), když pro dotčené týdny neexistuje týdenní záznam (`trips.temperature_source` / `water_temperature_source` říká, který zdroj se použil). |
| **Teplota mořské vody (SST) odděleně** | Nullable `water_temp_*` na `weather_records` / `weather_climate_months` + cache na `trips`; ingest jen u oblastí s párovým marine bodem (`marine_latitude` / `marine_longitude`). Scope v1: jen moře/oceán (Open-Meteo Marine) — jezera a bazény mimo. Katalogový filtr vyžaduje vyplněnou water cache. Viz [Teplota mořské vody](weather-and-climate.md#teplota-mořské-vody-sst). |
| **Geo-fallback oblastí počasí** | Hierarchie `postal_code` → `locality` → `subdivision` bez `parent_id` — odvozená z ISO polí. `places.weather_region_id` ukládá nejjemnější oblast; při chybějícím `weather_records` lookup za běhu zkusí rodiče (město → stát). Platí pro teplotu (včetně SST), balení i robotaxi; klimatický fallback zůstává jen pro teplotní cache. Viz [Hierarchie oblastí a geo-fallback](weather-and-climate.md#hierarchie-oblastí-a-geo-fallback). |
| **Cestovní požadavky odděleně od balení** | Dokumenty a formality jsou trip-level cache odvozená z geografie míst (`places.country_code`), ne z počasí. UI zobrazuje dva bloky: „Co si sbalit“ (oblečení) a „Dokumenty a formality“ (cestovní požadavky). |
| **Dojezd odděleně od délky programu** | `trips.total_duration_minutes` = délka aktivit a dopravy v itineráři. Katalogový filtr „kam za N hodin z místa X“ používá `travel_time_estimates` (place → place) a cache `trips.destination_place_id` / `outbound_transport_mode` (režim z itineráře). Doména je oddělená od `weather_regions` i robotaxi service areas — bez samostatné tabulky hubů. |
| **Robotaxi doména odděleně** | Provideři (`robotaxi_providers`), modely vozidel, servisní oblasti a weather advisories jsou samostatná doména navázaná na `transit_details`. Cache `segment_robotaxi_advisories` / `trip_robotaxi_advisories` je oddělená od balení i cestovních požadavků. |
| **DB constrainty + aplikační validace** | Základní invarianty (čas segmentu, ceny, rozsahy a konzistence počasí, věk), FK, partial unique indexy a nepřekrývání segmentů vynucuje PostgreSQL. Workflow pravidla a doménové cross-table validace (`segment_kind` ↔ `transit_details`, role, cache přepočty) zůstávají v aplikaci. |
| **Tři vrstvy měny** | Segment ukládá **místní** cenu (`local_price_*` — co platíte na místě), **domácí/plánovací** přepočet (`home_price_amount` v `trips.home_currency` — volba autora výletu) a **USD kanon** (`price_usd` — srovnání a filtry veřejného katalogu). Veřejný katalog filtruje a řadí vždy podle `total_cost_usd`; rozpočet výletu zobrazuje `total_cost_home` v `home_currency`. |
| **Denormalizace s účelem** | `trips.created_by` = audit / rychlý lookup; `trip_members` = jediný zdroj pravdy pro oprávnění. Totéž pro agregace na `trips` (`total_cost_usd`, `total_cost_home`, `total_duration_minutes`, `destination_place_id` / `outbound_transport_mode`, `recommended_age_min` / `recommended_age_max`, `max_difficulty`, `temp_min_c` / `temp_max_c`, `feels_like_min_c` / `feels_like_max_c`, `temperature_source`, `water_temp_min_c` / `water_temp_max_c`, `water_temperature_source`, `rating_avg` / `rating_count`, `packing_computed_at` + `trip_packing_items`, `travel_requirements_computed_at` + `trip_travel_requirements`, `robotaxi_advisories_computed_at` + `trip_robotaxi_advisories`) a `exchange_rate_local_to_usd` / `exchange_rate_local_to_home` na segmentu; na `places` cache `review_rating_avg` / `review_rating_count` z `place_reviews` (odděleně od externího `places.rating`). Filtrovaný veřejný katalog vyžaduje u teploty a náročnosti vyplněnou cache (`feels_like_*` pro primární teplotní filtr, `max_difficulty` NOT NULL); u filtru teploty vody `water_temp_*` NOT NULL; u filtru dojezdu párovou cache destinace/režimu + odhad v `travel_time_estimates` (nebo stejné místo); u věku znamená `NULL` „bez limitu“. |
| **UUID identifikátory** | Všechny primární klíče jsou typu `UUID` (`gen_random_uuid()`). |
| **Auditní pole** | Každá **entitní** tabulka má `created_at TIMESTAMPTZ NOT NULL DEFAULT now()` a `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`; `updated_at` udržuje automatický trigger. Cache tabulky (`segment_packing_items`, `segment_packing_item_sources`, `trip_packing_items`, `trip_packing_item_sources`, `trip_travel_requirements`, `trip_travel_requirement_sources`, `segment_robotaxi_advisories`, `trip_robotaxi_advisories`) a junction tabulky `clothing_rule_*` / `robotaxi_advisory_rule_*` auditní pole nemají. |
| **Autorský obsah segmentu** | `segments.title`, `segments.description` a galerie `segment_images` jsou autorský obsah v jazyce autora výletu — oddělený od strukturálních dat (čas, místa, cena) a neagreguje se na `trips`. |
| **Hard delete výletů** | Výlet se maže fyzicky (`DELETE FROM trips`); CASCADE smaže `trip_members`, `trip_reviews` (+ `trip_review_media`), `trip_packing_items`, `trip_packing_item_sources`, `trip_travel_requirements`, `trip_travel_requirement_sources`, `segment_robotaxi_advisories`, `trip_robotaxi_advisories`, `segments`, `segment_images`, `segment_packing_items` (+ `segment_packing_item_sources`) a `transit_details`. Skrytí z katalogu řeší `status` + `visibility`. Hard delete místa (`DELETE FROM places`) CASCADE smaže `place_reviews` (+ `place_review_media`) a řádky `travel_time_estimates`, kde je místo origin nebo destinace; `trips.destination_place_id` se nastaví na `NULL` (SET NULL). |
| **Lokalizace na FE** | DB ukládá stabilní slugy a enum hodnoty; přeložené labely (oblečení, cestovní požadavky, počasí, klima / suitability měsíců, náročnost, kategorie, přijímané platby místa, last-mile robotaxi access, robotaxi provideři, upozornění) řeší frontend podle `users.locale`. UGC texty recenzí (`trip_reviews.body`, `place_reviews.body`, captiony médií) zůstávají v jazyce autora — bez systémového překladu v DB. |

---

## ENUM typy

| Typ | Hodnoty |
|---|---|
| `trip_status` | `draft`, `published` |
| `trip_visibility` | `private`, `unlisted`, `public` |
| `member_role` | `admin`, `editor`, `viewer` |
| `segment_difficulty` | `easy`, `moderate`, `hard`, `extreme` — pořadí deklarace = rostoucí náročnost (pro `MAX()` agregaci na `trips.max_difficulty`) |
| `segment_kind` | `accommodation`, `activity`, `transit` |
| `transport_mode` | `walk`, `robotaxi`, `own_car`, `car_rental`, `rideshare`, `bus`, `tram`, `metro`, `train`, `plane`, `helicopter`, `boat`, `bicycle`, `bike_share`, `scooter`, `motorcycle`, `cable_car`, `other` |
| `robotaxi_service_status` | `waitlist`, `public`, `suspended` — stav servisní oblasti providera |
| `robotaxi_advisory_severity` | `info`, `warning` — závažnost robotaxi upozornění (počasí × provoz) |
| `precipitation_intensity` | `none`, `drizzle`, `light`, `moderate`, `heavy`, `storm` |
| `sky_condition` | `clear`, `mostly_sunny`, `partly_cloudy`, `variable`, `mostly_cloudy`, `cloudy`, `overcast` — pořadí deklarace = rostoucí zataženost (pro `MAX()` agregaci klimatu); `variable` leží uprostřed škály, aby nepřebilo `overcast` |
| `wind_force` | `calm`, `light`, `moderate`, `fresh`, `strong`, `gale`, `storm` |
| `fog_condition` | `none`, `haze`, `mist`, `fog`, `dense_fog` |
| `clothing_category` | `outerwear`, `layering`, `footwear`, `accessories`, `protection` |
| `packing_source` | `weather`, `segment` — proč byla položka doporučena (cache balení na úrovni výletu) |
| `travel_requirement_source` | `trip_geography`, `manual` — proč byla položka doporučena (cache cestovních požadavků na úrovni výletu) |
| `temperature_source` | `weather_records`, `climate` — odkud pochází teplotní cache na `trips` (vzduchová/pocitová i SST přes `water_temperature_source`); `weather_records` zahrnuje i rodičovskou oblast přes geo-fallback; `climate` = alespoň jeden dotčený týden se dopočítal z `weather_climate_months` |
| `weather_region_type` | `postal_code`, `locality`, `subdivision`, `custom` — úroveň geo oblasti pro agregaci počasí; `subdivision` = stát/kraj/provincie dle ISO 3166-2; runtime geo-fallback: postal → locality → subdivision |
| `review_media_kind` | `image`, `video` — typ média v galerii uživatelské recenze |
| `place_accepted_payments` | `card`, `cash`, `card_and_cash` — jak se na místě platí; na sloupci `places.accepted_payments` je `NULL` = neznámé / nezadáno |
| `place_robotaxi_access` | `direct`, `via_access_point`, `not_accessible` — last-mile dostupnost robotaxi k POI; na sloupci `places.robotaxi_access` je `NULL` = neznámé / nezadáno; nezávislé na `provider_service_areas` |

---

## Datový model — klíčové vazby

```
users ──< trip_members >── trips ──< segments (segment_kind) ──< segment_images
  │                         │              │
  ├── created_by (RESTRICT) ┘              ├── segment_packing_items >── clothing_items
  │                         │              │       └── segment_packing_item_sources
  │                         │              ├── segment_robotaxi_advisories >── robotaxi_advisory_items
  │                         │              └── transit_details (1:1) ──> robotaxi_providers
  │                         │                      ├── pickup/dropoff_zone → places
  │                         │                      └── vehicle_model_id → robotaxi_vehicle_models
  │                         ├── trip_packing_items >── clothing_items
  │                         │       └── trip_packing_item_sources
  │                         ├── trip_robotaxi_advisories >── robotaxi_advisory_items
  │                         ├── trip_travel_requirements >── travel_requirement_items
  │                         │       └── trip_travel_requirement_sources
  │                         ├── trip_reviews ──< trip_review_media
  │                         │       └── user_id (RESTRICT)
  │                         └── home_currency, total_cost_usd, total_cost_home, total_duration_minutes,
  │                             destination_place_id, outbound_transport_mode, recommended_age_min/max,
  │                             max_difficulty, temp_min_c/max_c, feels_like_min_c/max_c, temperature_source,
  │                             water_temp_min_c/max_c, water_temperature_source, rating_avg/rating_count,
  │                             packing_computed_at, travel_requirements_computed_at, robotaxi_advisories_computed_at (cache)
  │
  └── place_reviews ──< place_review_media
          │
          └── place_id → places

place_categories ──< places ── (start_place_id / end_place_id on segments; trips.destination_place_id)
                         │
                         ├── weather_region_id → weather_regions ──< weather_records
                         │                                    └──< weather_climate_months
                         ├── country_code (ISO 3166-1 alpha-2)
                         ├── accepted_payments (card / cash / card_and_cash; NULL = neznámé)
                         ├── robotaxi_access (direct / via_access_point / not_accessible; NULL = neznámé)
                         ├── robotaxi_access_place_id → places (last-mile dropoff; SET NULL)
                         ├── robotaxi_approach_walk_meters
                         ├── external_source / external_place_id (identita importu; partial UNIQUE)
                         ├── rating (externí Maps)
                         └── review_rating_avg / review_rating_count (cache z place_reviews)

travel_time_estimates (origin_place_id / destination_place_id × transport_mode)

robotaxi_providers ──< robotaxi_vehicle_models
                 └──< provider_service_areas

robotaxi_advisory_items ──< robotaxi_advisory_rules ──< robotaxi_advisory_rule_* (junction)

clothing_items ──< clothing_rules ──< clothing_rule_* (junction tabulky podmínek)

travel_requirement_items ──< travel_requirement_rules
```

### Příklad timeline

```
Trip: „Praha víkend"
│
├── Hotel — noc + check-out pá 18:00–so 10:00 (segment_kind: accommodation)
├── Chůze → pick-up zóna — so 10:00–10:15  (segment_kind: transit)
├── Robotaxi → Karlštejn — so 10:15–10:45  (segment_kind: transit)
├── Chůze → hrad — so 10:45–11:00         (segment_kind: transit)
├── Prohlídka hradu — so 11:00–13:00       (segment_kind: activity)
└── Robotaxi → hotel — so 13:00–13:30      (segment_kind: transit)
```

Typ segmentu určuje sloupec `segment_kind`; `end_place_id`, `transit_details` a kategorie místa musí být s ním konzistentní — viz [Sémantika segmentů](segments.md#sémantika-segmentů). Sousedící segmenty bez mezery (např. ubytovací úsek končí v 10:00, chůze začíná v 10:00) jsou validní — intervaly jsou polootevřené `[start, end)`.

