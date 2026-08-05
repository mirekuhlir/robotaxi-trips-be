# Implementation notes

[← Přehled schématu](README.md)

Cross-cutting přehled agregací na úrovni výletu. Doménové poznámky jsou u příslušných souborů — viz [index níže](#doménové-poznámky).

### Agregace na úrovni `trips`

Sloupce `total_cost_usd`, `total_cost_home`, `total_duration_minutes`, `destination_place_id`, `outbound_transport_mode`, `recommended_age_min`, `recommended_age_max`, `max_difficulty`, `temp_min_c`, `temp_max_c`, `feels_like_min_c`, `feels_like_max_c`, `temperature_source`, `rating_avg`, `rating_count`, `packing_computed_at`, `travel_requirements_computed_at`, `robotaxi_advisories_computed_at` a tabulky `trip_packing_items`, `trip_packing_item_sources`, `segment_packing_items`, `segment_packing_item_sources`, `trip_travel_requirements`, `trip_travel_requirement_sources`, `segment_robotaxi_advisories`, `trip_robotaxi_advisories` jsou denormalizované cache hodnoty. Agregace ze segmentů aktualizuj aplikační vrstvou při INSERT/UPDATE/DELETE na `segments` (v téže transakci); `rating_avg` / `rating_count` při změně `trip_reviews` (viz [Recenze](users-and-trips.md#recenze)):

- **`total_cost_usd`** — read-only cache; součet `price_usd` ze **všech** segmentů výletu (včetně ubytování). Při INSERT/UPDATE segmentu: načíst kurz `local → USD` (viz [Kurz měny](segments.md#kurz-měny)) → uložit `exchange_rate_local_to_usd` → spočítat `price_usd = ROUND(local_price_amount × exchange_rate_local_to_usd, 2)`. Změna kurzu při editaci segmentu je očekávaná u draft výletů; `exchange_rate_local_to_usd` umožňuje audit, proč se cena v USD liší. Prázdný výlet → `0`. **Veřejný katalog filtruje a řadí vždy podle tohoto sloupce.**
- **`total_cost_home`** — read-only cache; součet `home_price_amount` ze **všech** segmentů výletu v měně `trips.home_currency`. Při INSERT/UPDATE segmentu: načíst kurz `local → home_currency` (viz [Kurz měny](segments.md#kurz-měny)) → uložit `exchange_rate_local_to_home` → spočítat `home_price_amount`. Při změně `trips.home_currency` přepočítat **všechny** segmenty výletu a znovu agregovat `total_cost_home` v jedné transakci. Prázdný výlet → `0`. Slouží pro rozpočet a zobrazení výletu autorovi — **nepoužívat** pro veřejný katalog.
- **`total_duration_minutes`** — read-only cache; součet délky (`end_time - start_time`) segmentů **kromě ubytování** (`segment_kind = accommodation`). Prázdný výlet → `0`. **Není** doba dojezdu z vybraného místa — ta žije v `travel_time_estimates` + cache destinace níže.
- **`destination_place_id` / `outbound_transport_mode`** — read-only cache pro katalogový filtr dojezdu; párová (`NULL`/`NULL` nebo obě vyplněné). Odvození ze segmentů — viz [Cache destinace a outbound režimu](travel-times.md#cache-destinace-a-outbound-režimu-na-trips).
- **`recommended_age_min`** — nejvyšší požadovaný min. věk mezi aktivitami: `MAX(segments.recommended_age_min)` přes aktivity (`segment_kind = activity`) s neprázdným `recommended_age_min`; pokud žádná aktivita nemá min, výsledek `NULL`
- **`recommended_age_max`** — nejnižší horní hranice věku mezi aktivitami: `MIN(segments.recommended_age_max)` přes aktivity s neprázdným `recommended_age_max`; pokud žádná aktivita nemá max, výsledek `NULL`
- **`max_difficulty`** — read-only cache; `MAX(segments.difficulty)` přes aktivity (`segment_kind = activity`) s neprázdným `difficulty`; pokud žádná aktivita nemá náročnost, výsledek `NULL`. Pořadí enumu `segment_difficulty` musí odpovídat rostoucí náročnosti (`easy` < `moderate` < `hard` < `extreme`) — PostgreSQL `MAX()` na enumu používá pořadí deklarace typu.
- **`temp_min_c`** / **`temp_max_c`** — read-only cache **vzduchové** teploty; viz [Lookup počasí podle typu segmentu](weather-and-climate.md#lookup-počasí-podle-typu-segmentu). Agregace ze **všech** typů segmentů (`accommodation`, `transit`, `activity`) podle pravidel lookup. `temp_min_c = MIN(všechny vyřešené min)`, `temp_max_c = MAX(všechny vyřešené max)`. Deduplikace podle **vyřešeného** `(resolved_weather_region_id, week_start)` — regionu, ze kterého skutečně přišel záznam (včetně rodiče přes [geo-fallback](weather-and-climate.md#hierarchie-oblastí-a-geo-fallback)). Chybí-li týdenní záznam v celém geo-řetězci, kombinace se dopočítá z klimatu ve stejném řetězci (viz [Fallback na klima](weather-and-climate.md#fallback-na-klima)). Žádný vyřešitelný zdroj → oba sloupce `NULL`.
- **`feels_like_min_c`** / **`feels_like_max_c`** — read-only cache **pocitové** teploty (viz [Pocitová teplota](weather-and-climate.md#pocitová-teplota)). Stejný lookup, geo-fallback, fallback na klima i deduplikace jako u `temp_*`. Efektivní min/max z každého záznamu dle kroku 4 lookupu (fallback na vzduchovou, pokud pocitová chybí). `feels_like_min_c = MIN(...)`, `feels_like_max_c = MAX(...)`. Žádný vyřešitelný zdroj → oba sloupce `NULL` → výlet se nezobrazí ve filtrovaném katalogu dle teploty (primární filtr je pocitová; viz [Teplota výletu](weather-and-climate.md#teplota-výletu)).
- **`temperature_source`** — read-only cache; `weather_records` když všechny přispívající kombinace měly týdenní záznam (včetně rodičovské oblasti), `climate` když se alespoň jedna dopočítala z `weather_climate_months` (včetně klimatu rodiče), `NULL` když je teplotní cache prázdná. Nastavuje se v témže průchodu jako `temp_*` / `feels_like_*`.
- **Cache balení** — přepočet podle sekce [Doporučené oblečení](packing.md#doporučené-oblečení): `DELETE` + `INSERT` do `segment_packing_items` + `segment_packing_item_sources`, agregace do `trip_packing_items` + `trip_packing_item_sources` (zdroje jako union segmentových), nastavení `trips.packing_computed_at`
- **Cache cestovních požadavků** — přepočet podle sekce [Cestovní požadavky](travel-requirements.md#cestovní-požadavky): geo položky z `travel_requirement_rules`, merge s ručními položkami (`source = manual`), nastavení `trips.travel_requirements_computed_at`
- **Cache robotaxi upozornění** — přepočet podle sekce [Robotaxi upozornění](robotaxi.md#robotaxi-upozornění): `DELETE` + `INSERT` do `segment_robotaxi_advisories`, agregace do `trip_robotaxi_advisories`, nastavení `trips.robotaxi_advisories_computed_at`
- **`rating_avg` / `rating_count`** — read-only cache z `trip_reviews`; viz [Recenze](users-and-trips.md#recenze). Žádné recenze → `rating_count = 0`, `rating_avg = NULL`.

**Lookup počasí pro teplotu a balení:** viz kanonický algoritmus v [Lookup počasí podle typu segmentu](weather-and-climate.md#lookup-počasí-podle-typu-segmentu) včetně [geo-fallbacku](weather-and-climate.md#hierarchie-oblastí-a-geo-fallback). Pro agregaci `temp_*` i `feels_like_*` na `trips` použít krok 4 (fallback teplot) a deduplikovat stejný `(resolved_weather_region_id, week_start)` — vzduchová: `temp_min_c = MIN`, `temp_max_c = MAX`; pocitová: `feels_like_min_c = MIN`, `feels_like_max_c = MAX`. Kombinace bez týdenního záznamu v geo-řetězci se pro teplotu (a jen pro ni) dopočítá z `weather_climate_months` ve stejném řetězci.

Přepočet vzduchové i pocitové teploty probíhá i při změně `weather_records` pro oblasti/týdny dotčené výletem **nebo jejich rodiče v geo-řetězci** (stejný aplikační handler / job jako u cache balení — lze počítat v jednom průchodu). Nově dodaný týdenní záznam na jemnější oblasti nahradí rodičovský i klimatický odhad, takže se přepočtem může `temperature_source` změnit z `climate` na `weather_records`. Přepočet je nutný i po sync `weather_climate_months`, pokud oblast (nebo rodič použitý jako fallback) dosud klima neměla.

Výlet bez aktivit nebo bez věkových limitů → oba sloupce věku `NULL`. Výlet bez aktivit s náročností → `max_difficulty` `NULL`. Výlet bez určitelné destinace / outbound režimu → `destination_place_id` i `outbound_transport_mode` `NULL`. Přepočet ceny, délky, destinace/outbound režimu, věku, náročnosti a teploty probíhá ve **stejné transakci** při INSERT/UPDATE/DELETE segmentu, změně místní ceny/měny, změně `trips.home_currency`, času, `difficulty`, `segment_kind` nebo `transit_details.transport_mode`.

**Příklad agregace ceny a délky** (výlet „Praha víkend", `trips.home_currency = 'CZK'`):

| Segment | `local_price_*` | `home_price_amount` | `price_usd` | délka (min) | započítáno do duration |
|---|---|---|---|---|---|
| Hotel (accommodation, noc + check-out pá 18:00 – so 10:00) | 2000 CZK | 2000 | 80 | 960 | ne |
| Chůze → pick-up (transit) | 0 CZK | 0 | 0 | 15 | ano |
| Robotaxi (transit) | 300 CZK | 300 | 12 | 30 | ano |
| Chůze → hrad (transit) | 0 CZK | 0 | 0 | 15 | ano |
| Prohlídka hradu (activity) | 375 CZK | 375 | 15 | 120 | ano |
| Robotaxi zpět (transit) | 300 CZK | 300 | 12 | 30 | ano |
| → **`trips` cache** | | **`2975` CZK** | **`119` USD** | **210** | |

Poznámka: `home_price_amount` = `local_price_amount`, protože všechny segmenty jsou v CZK a `home_currency` je také CZK (kurz `1.0`). U výletu v zahraničí by místní měna (např. THB) a `home_currency` (CZK) typicky nesouhlasily — viz [Kurz měny](segments.md#kurz-měny).

**Příklad agregace věku** (výlet „Praha víkend"):

| Aktivita | `recommended_age_min` | `recommended_age_max` |
|---|---|---|
| Prohlídka hradu | 6 | NULL |
| → **`trips` cache** | **6** | **NULL** |

**Příklad agregace náročnosti** (výlet „Praha víkend"):

| Aktivita | `difficulty` |
|---|---|
| Prohlídka hradu | `moderate` |
| → **`trips.max_difficulty`** | **`moderate`** |

**Příklad agregace teploty** (výlet „Praha víkend") — lookup per `(resolved_weather_region_id, week_start)` po geo-fallbacku; u transitu/aktivity s `end_place_id` každý region zvlášť. Stejné kombinace slouží pro vzduchovou i pocitovou cache (v příkladu pocitová nižší kvůli větru):

| Segment | region | `temp_min_c` | `temp_max_c` | `feels_like_min_c` | `feels_like_max_c` | poznámka |
|---|---|---|---|---|---|---|
| Hotel (accommodation, noční úsek) | Praha | 8 | 18 | 7 | 17 | |
| Chůze → pick-up (transit) | Praha | 8 | 18 | 7 | 17 | start_place |
| Robotaxi → Karlštejn (transit) | Praha | 8 | 18 | 7 | 17 | start_place |
| Robotaxi → Karlštejn (transit) | Střední Čechy | 5 | 16 | 3 | 14 | end_place |
| Chůze → hrad (transit) | Střední Čechy | 5 | 16 | 3 | 14 | start + end (stejný region) |
| Prohlídka hradu (activity) | Střední Čechy | 5 | 16 | 3 | 14 | |
| Robotaxi zpět (transit) | Střední Čechy | 5 | 16 | 3 | 14 | start_place |
| Robotaxi zpět (transit) | Praha | 8 | 18 | 7 | 17 | end_place |
| → deduplikace `(region × týden)` | | | | | | |
| → **`trips` cache** | | **5** | **18** | **3** | **17** | `MIN` / `MAX` přes unikátní kombinace |

Všechny kombinace v příkladu mají týdenní záznam, takže `temperature_source = 'weather_records'`. Kdyby pro některý týden záznam chyběl, dosadí se klimatický normál příslušného měsíce a cache dostane `temperature_source = 'climate'`.

U delších pobytů se ubytování rozdělí na nepřekrývající se noční/check-in/check-out úseky. Každý úsek, který protíná hranici lokálního ISO týdne, započítá oba dotčené týdny pro danou oblast.


---

## Doménové poznámky

| Doména | Soubor |
|---|---|
| Status, visibility, mazání; Recenze | [users-and-trips.md](users-and-trips.md) |
| Geografické dotazy | [places.md](places.md#geografické-dotazy) |
| Sémantika segmentů, obsah, cena, věk, `transit_details` | [segments.md](segments.md) |
| Teplota výletu; Počasí a oblasti | [weather-and-climate.md](weather-and-climate.md) |
| Doporučené oblečení | [packing.md](packing.md#doporučené-oblečení) |
| Cestovní požadavky | [travel-requirements.md](travel-requirements.md#cestovní-požadavky) |
| Dojezd mezi městy; filtr katalogu | [travel-times.md](travel-times.md) |
| Robotaxi upozornění; Servisní oblasti | [robotaxi.md](robotaxi.md) |
| DB constrainty a indexy | [constraints-and-indexes.md](constraints-and-indexes.md) |
