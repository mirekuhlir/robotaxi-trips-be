# Weather & climate

[← Přehled schématu](README.md)

## Tabulky

### `weather_regions`

Geografická oblast pro agregaci týdenního počasí. Více míst v téže oblasti sdílí stejné záznamy. Identifikace oblasti je **administrativní** (ISO kódy), ne geometrická — souřadnice středu slouží pro dotaz na weather API a jako poslední fallback při auto-párování.

Oblasti typu `custom` **záměrně nemají unique constraint** (na rozdíl od partial unique indexů pro `postal_code` / `locality` / `subdivision`) — zakládají a spravují se ručně a jejich unikátnost hlídá admin proces, ne DB.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `region_type` | weather_region_type, NOT NULL | Úroveň oblasti (`postal_code`, `locality`, `subdivision`, `custom`) |
| `name` | VARCHAR, NOT NULL | Zobrazovaný název, např. `Praha`, `San Francisco, CA 94102`, `Karlštejn okolí` |
| `country_code` | CHAR(2), NOT NULL | ISO 3166-1 alpha-2 (`US`, `CZ`) |
| `subdivision_code` | VARCHAR, nullable | ISO 3166-2 bez prefixu země (`CA`, `20`); povinné pro `region_type = locality` i `subdivision` kvůli jednoznačné identifikaci oblasti |
| `subdivision_name` | VARCHAR, nullable | Lidsky čitelný název subdivize (`California`, `Středočeský kraj`) |
| `locality` | VARCHAR, nullable | Město / obec (`San Francisco`, `Praha`) |
| `postal_code` | VARCHAR, nullable | Normalizované PSČ/ZIP bez mezer (`94102`, `11000`) |
| `timezone` | VARCHAR, NOT NULL | IANA časová zóna oblasti (`Europe/Prague`, `America/Los_Angeles`) pro výpočet lokálního ISO týdne počasí |
| `center_latitude` | NUMERIC(9, 6), NOT NULL | Střed oblasti — weather API a geo fallback (`-90` až `90`) |
| `center_longitude` | NUMERIC(9, 6), NOT NULL | Střed oblasti — weather API a geo fallback (`-180` až `180`) |
| `marine_latitude` | NUMERIC(9, 6), nullable | Bod v moři/oceánu pro SST ingest (Open-Meteo Marine); párový s `marine_longitude` — viz [Teplota mořské vody](#teplota-mořské-vody-sst) |
| `marine_longitude` | NUMERIC(9, 6), nullable | Bod v moři/oceánu pro SST ingest; párový s `marine_latitude` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Marine bod:** obě `NULL` (= oblast nemá mořskou vodu — typicky vnitrozemí) **nebo** obě vyplněné. Nepoužívej slepě `center_*` — centroid města často leží ve vnitrozemí a Marine API vrátí prázdno nebo špatnou buňku. Admin / sync job nastaví nejbližší mořskou buňku u pobřežních oblastí. SST ingest běží **jen** když je pár nastavený.

**Seed data (příklad):**

| name | region_type | country_code | subdivision_code | locality | postal_code | timezone |
|---|---|---|---|---|---|---|
| California | subdivision | US | CA | | | America/Los_Angeles |
| Los Angeles, CA | locality | US | CA | Los Angeles | | America/Los_Angeles |
| San Francisco, CA 94102 | postal_code | US | CA | San Francisco | 94102 | America/Los_Angeles |

Při zakládání `locality` (nebo `postal_code`) zajisti odpovídající rodičovskou `subdivision` pro stejné `(country_code, subdivision_code)` — bez ní [geo-fallback](#hierarchie-oblastí-a-geo-fallback) nemá kam spadnout. Subdivision se plní vlastním syncem (centroid `center_*`), ne agregací měst.

### `weather_records`

Týdenní souhrn počasí pro danou oblast. Jeden záznam = jeden lokální kalendářní týden (pondělí–neděle) v časové zóně `weather_regions.timezone`.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `weather_region_id` | UUID, FK → weather_regions | ON DELETE CASCADE |
| `week_start` | DATE | Pondělí daného týdne (lokální ISO týden dle `weather_regions.timezone`) |
| `temp_avg_c` | NUMERIC(4, 1) | Průměrná **vzduchová** teplota ve °C (API vždy °C; FE dle `users.unit_system`) |
| `temp_min_c` | NUMERIC(4, 1), nullable | Minimální vzduchová teplota ve °C |
| `temp_max_c` | NUMERIC(4, 1), nullable | Maximální vzduchová teplota ve °C |
| `humidity_avg_pct` | NUMERIC(4, 1), nullable | Průměrná relativní vlhkost vzduchu v % (0–100) — bez přepočtu jednotek |
| `feels_like_avg_c` | NUMERIC(4, 1), nullable | Průměrná pocitová teplota ve °C (*apparent temperature*; viz [Pocitová teplota](#pocitová-teplota)) |
| `feels_like_min_c` | NUMERIC(4, 1), nullable | Minimální pocitová teplota ve °C |
| `feels_like_max_c` | NUMERIC(4, 1), nullable | Maximální pocitová teplota ve °C |
| `water_temp_avg_c` | NUMERIC(4, 1), nullable | Průměrná teplota povrchu moře/oceánu (SST) ve °C; `NULL` = oblast bez marine bodu nebo API bez dat — viz [Teplota mořské vody](#teplota-mořské-vody-sst) |
| `water_temp_min_c` | NUMERIC(4, 1), nullable | Minimální SST ve °C za týden |
| `water_temp_max_c` | NUMERIC(4, 1), nullable | Maximální SST ve °C za týden |
| `sky_condition` | sky_condition | Dominantní oblačnost / sluneční charakter týdne |
| `sunshine_hours` | NUMERIC(5, 1), nullable | Součet hodin slunečního svitu za týden |
| `precipitation_intensity` | precipitation_intensity | Dominantní intenzita srážek (`none` = bez deště) |
| `rain_mm` | NUMERIC(6, 1) | Celkový déšť za týden v mm vody (0 = bez deště); výchozí `0`. API vždy mm; FE dle `users.unit_system` (mm / in) |
| `rainy_days` | SMALLINT | Počet deštivých dní v týdnu (0–7); výchozí `0` |
| `wind_force` | wind_force | Dominantní síla větru za týden |
| `wind_speed_avg_ms` | NUMERIC(4, 1), nullable | Průměrná rychlost větru v m/s. API vždy m/s; FE dle `users.unit_system` (m/s / mph) |
| `wind_speed_max_ms` | NUMERIC(4, 1), nullable | Maximální náraz větru v m/s |
| `wind_direction_avg_deg` | SMALLINT, nullable | Průměrný směr větru ve stupních (0–359, 0 = sever) — bez přepočtu jednotek |
| `fog_condition` | fog_condition | Dominantní mlha / viditelnost za týden (`none` = bez mlhy) |
| `fog_days` | SMALLINT | Počet mlhavých dní v týdnu (0–7); výchozí `0` |
| `visibility_avg_m` | INTEGER, nullable | Průměrná viditelnost v metrech (nižší = horší kvůli mlze). API vždy metry; FE dle `users.unit_system` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Omezení:** `UNIQUE (weather_region_id, week_start)` — v oblasti maximálně jeden záznam na týden.

**Konzistence srážek** (DB CHECK; aplikace validuje před zápisem pro lepší chybovou hlášku): `precipitation_intensity = 'none'` ⇒ `rain_mm = 0` a `rainy_days = 0`; opačný směr (`rain_mm > 0` nebo `rainy_days > 0` ⇒ `precipitation_intensity` nesmí být `none`) je kontrapozice téhož pravidla — pokrývá ho stejný CHECK.

**Konzistence mlhy** (DB CHECK; aplikace validuje před zápisem pro lepší chybovou hlášku): `fog_condition = 'none'` ⇒ `fog_days = 0`; opačný směr (`fog_days > 0` ⇒ `fog_condition` nesmí být `none`) pokrývá stejný CHECK.

### `weather_climate_months`

Měsíční klimatický normál pro danou oblast. Jeden záznam = typický kalendářní měsíc (leden–prosinec) — **ne** konkrétní týden ani konkrétní rok. Dimenze záměrně zrcadlí `weather_records`, aby FE a skórování „ideální měsíc“ používaly stejný slovník. Zdroj pravdy pro přehled „kdy jet“; týdenní prognóza/historie zůstává v `weather_records`.

| Sloupec | Typ | Popis |
|---|---|---|
| `id` | UUID, PK | |
| `weather_region_id` | UUID, FK → weather_regions | ON DELETE CASCADE |
| `month` | SMALLINT, NOT NULL | Kalendářní měsíc `1`–`12` (1 = leden) |
| `temp_avg_c` | NUMERIC(4, 1), NOT NULL | Typická průměrná **vzduchová** teplota ve °C |
| `temp_min_c` | NUMERIC(4, 1), nullable | Typická minimální vzduchová teplota ve °C |
| `temp_max_c` | NUMERIC(4, 1), nullable | Typická maximální vzduchová teplota ve °C |
| `humidity_avg_pct` | NUMERIC(4, 1), nullable | Typická průměrná relativní vlhkost vzduchu v % (0–100) |
| `feels_like_avg_c` | NUMERIC(4, 1), nullable | Typická průměrná pocitová teplota ve °C (viz [Pocitová teplota](#pocitová-teplota)) |
| `feels_like_min_c` | NUMERIC(4, 1), nullable | Typická minimální pocitová teplota ve °C |
| `feels_like_max_c` | NUMERIC(4, 1), nullable | Typická maximální pocitová teplota ve °C |
| `water_temp_avg_c` | NUMERIC(4, 1), nullable | Typická průměrná teplota povrchu moře/oceánu (SST) ve °C; `NULL` u vnitrozemí — viz [Teplota mořské vody](#teplota-mořské-vody-sst) |
| `water_temp_min_c` | NUMERIC(4, 1), nullable | Typické minimální SST ve °C |
| `water_temp_max_c` | NUMERIC(4, 1), nullable | Typické maximální SST ve °C |
| `sky_condition` | sky_condition, NOT NULL | Dominantní typická oblačnost / sluneční charakter měsíce |
| `sunshine_hours` | NUMERIC(5, 1), nullable | Typický součet hodin slunečního svitu za měsíc |
| `precipitation_intensity` | precipitation_intensity, NOT NULL | Dominantní typická intenzita srážek (`none` = bez deště) |
| `rain_mm` | NUMERIC(6, 1), NOT NULL | Typický úhrn srážek za měsíc v mm vody (0 = bez deště); výchozí `0`. API vždy mm; FE dle `users.unit_system` |
| `rainy_days` | SMALLINT, NOT NULL | Typický počet deštivých dní v měsíci (0–31); výchozí `0` |
| `wind_force` | wind_force, NOT NULL | Dominantní typická síla větru za měsíc |
| `fog_condition` | fog_condition, NOT NULL | Dominantní typická mlha za měsíc (`none` = bez mlhy) |
| `fog_days` | SMALLINT, NOT NULL | Typický počet mlhavých dní v měsíci (0–31); výchozí `0` |
| `created_at` | TIMESTAMPTZ | Výchozí `now()` |
| `updated_at` | TIMESTAMPTZ | Výchozí `now()`; Auto-trigger |

**Omezení:** `UNIQUE (weather_region_id, month)` — v oblasti maximálně jeden normál na kalendářní měsíc.

**Konzistence srážek** (DB CHECK; aplikace validuje před zápisem pro lepší chybovou hlášku): `precipitation_intensity = 'none'` ⇒ `rain_mm = 0` a `rainy_days = 0`; opačný směr (`rain_mm > 0` nebo `rainy_days > 0` ⇒ `precipitation_intensity` nesmí být `none`) pokrývá stejný CHECK.

**Konzistence mlhy** (DB CHECK; aplikace validuje před zápisem pro lepší chybovou hlášku): `fog_condition = 'none'` ⇒ `fog_days = 0`; opačný směr (`fog_days > 0` ⇒ `fog_condition` nesmí být `none`) pokrývá stejný CHECK.

Skóre „ideální měsíc“, labely a agregace na výlet se **neukládají** v této tabulce — viz [Ideální měsíce (klima)](#ideální-měsíce-klima).


---

## Poznámky k implementaci

### Teplota výletu

**Zdroj pravdy:** `weather_records` (region × lokální ISO týden dle `weather_regions.timezone`) — ne segment, ne trip. Vzduchová teplota (`temp_*`) i pocitová (`feels_like_*`, vlhkost + vítr) — viz [Pocitová teplota](#pocitová-teplota). Chybí-li záznam na oblasti místa, lookup nejdřív zkusí rodičovskou oblast ([geo-fallback](#hierarchie-oblastí-a-geo-fallback)); teprve pak se teplotní cache může dopočítat z klimatického normálu — viz [Fallback na klima](#fallback-na-klima).

**Trips (cache pro katalog):** Sloupce `temp_min_c` / `temp_max_c` (vzduchová) a `feels_like_min_c` / `feels_like_max_c` (pocitová) jsou agregovaná cache odvozená z počasí přes segmenty a místa — viz [Agregace na úrovni `trips`](implementation-notes.md#agregace-na-úrovni-trips). Sloupec `trips.temperature_source` říká, zda hodnoty pocházejí z týdenních dat, nebo z klimatu. Autor výletu je nevyplňuje ručně. Konzistentní s principem „počasí odděleně od balení“ — jde o odvozeninu pro vyhledávání, ne duplikaci celého `weather_records`.

**Katalog „teplota“ = pocitová.** Primární filtr používá `feels_like_min_c` / `feels_like_max_c` (NOT NULL povinné ve filtrovaném katalogu). Vzduchová `temp_*` zůstává pro UI (např. „3–17 °C pocitově, vzduch 5–18 °C“) a volitelný sekundární filtr. `NULL` u `feels_like_*` znamená, že výlet nemá vyřešitelné počasí **ani klima** (chybí `weather_region_id`, nebo oblast nemá ani jeden z obou zdrojů) — při chybějící pocitové v záznamu agregace fallbackuje na vzduchovou stejného směru, takže cache pocitové se obvykle vyplní i ze starších dat bez vlhkosti.

#### Fallback na klima

`weather_records` pokrývají archiv a prognózu na blízké týdny. Výlet naplánovaný na příští sezónu by bez fallbacku měl teplotní cache `NULL` a z filtrovaného katalogu by úplně vypadl. Proto se pro **teplotní cache na `trips`** po vyčerpání týdenních záznamů (včetně rodičů) používá klimatický normál:

1. Pro každou kombinaci `(place.weather_region_id, week_start)` z [lookupu](#lookup-počasí-podle-typu-segmentu) projít [geo-řetězec](#hierarchie-oblastí-a-geo-fallback) a zkusit `weather_records` na oblasti místa a případných rodičích — první hit vyhrává.
2. Chybí-li týdenní záznam v celém řetězci, projít **stejný geo-řetězec** a načíst `weather_climate_months` pro danou oblast / rodiče a **kalendářní měsíc, do kterého padá střed týdne** (čtvrtek daného ISO týdne v timezone **původní** oblasti místa). První hit vyhrává.
3. Chybí-li i klima v celém řetězci, kombinaci přeskočit.

Hodnota `trips.temperature_source`:

| Situace | `temperature_source` |
|---|---|
| Všechny přispívající kombinace z `weather_records` (včetně rodičovské oblasti) | `weather_records` |
| Alespoň jedna kombinace dopočítaná z klimatu (včetně klimatu rodiče) | `climate` |
| Žádná vyřešitelná kombinace (všechny teplotní sloupce `NULL`) | `NULL` |

Bez nové enum hodnoty ve v1: geo-fallback na rodičovský týdenní záznam se počítá jako `weather_records`, ne jako samostatný zdroj. Volitelně později „coarse region“ pro UI.

**Klimatický fallback platí jen pro teplotní cache na `trips`** (vzduchová, pocitová i SST — viz [Teplota mořské vody](#teplota-mořské-vody-sst)). Balení (`clothing_rules`) a robotaxi upozornění (`robotaxi_advisory_rules`) se dál vyhodnocují **výhradně proti `weather_records`** (včetně rodičovských přes geo-fallback) — měsíční normál nenese `visibility_avg_m`, rychlost větru ani denní variabilitu, takže by generoval nepřesná doporučení pro konkrétní termín. Přepočet teplotní cache tedy může doplnit klima i pro výlet, jehož balení zůstane prázdné.

FE může u `temperature_source = 'climate'` zobrazit teplotu jako orientační („dlouhodobý průměr pro dané období“) místo prognózy.

**Omezení v1:** týdenní granularita uvnitř týdne — víkend v jedné oblasti sdílí jeden týdenní souhrn; katalog nerozliší páteční vs. sobotní počasí bez `weather_records_daily` (backlog v2). Segment protínající hranici lokálního ISO týdne načítá **oba** dotčené týdny (viz lookup níže).

**Katalogové dotazy:**

```sql
-- Primární: výlety, kde i nejchladnější období bude pocitově alespoň X °C
SELECT * FROM trips
WHERE status = 'published' AND visibility = 'public'
  AND feels_like_min_c IS NOT NULL AND feels_like_min_c >= :min_teplota;

-- Primární: výlety s max. pocitovou teplotou do Y °C
SELECT * FROM trips
WHERE status = 'published' AND visibility = 'public'
  AND feels_like_max_c IS NOT NULL AND feels_like_max_c <= :max_teplota;

-- Sekundární (volitelné): filtr dle vzduchové teploty
SELECT * FROM trips
WHERE status = 'published' AND visibility = 'public'
  AND temp_min_c IS NOT NULL AND temp_min_c >= :min_vzduchova
  AND temp_max_c IS NOT NULL AND temp_max_c <= :max_vzduchova;

-- Teplota mořské vody (SST): výlety, kde i nejchladnější období má vodu alespoň X °C
SELECT * FROM trips
WHERE status = 'published' AND visibility = 'public'
  AND water_temp_min_c IS NOT NULL AND water_temp_min_c >= :min_voda;
```

Pro kombinované filtry s cenou, délkou, věkem a náročností viz [Cena a délka výletu](segments.md#cena-a-délka-výletu).


### Teplota mořské vody (SST)

**Scope v1:** jen **moře / oceán** (*sea surface temperature*). Jezera, přehrady a bazény jsou mimo scope — i když kategorie `beach_waterfront` zahrnuje i jezera/nábřeží. Teplota vody se ukládá a agreguje **jen** tam, kde má `weather_regions` vyplněný párový marine bod (`marine_latitude` / `marine_longitude`).

**Zdroj pravdy:** `weather_records.water_temp_*` (region × lokální ISO týden) — ne segment, ne trip. Měsíční normál v `weather_climate_months.water_temp_*` slouží pro blok „Kdy jet“ a jako fallback trip cache.

**Ingest (preferovaný):**

- Týdenní / forecast: [Open-Meteo Marine API](https://open-meteo.com/en/docs/marine-weather-api) — pole `sea_surface_temperature` na souřadnicích `marine_latitude` / `marine_longitude` (ne `center_*`). Agregovat na týdenní avg / min / max.
- Měsíční klima: agregace historického SST (Marine archive / Historical API s `sea_surface_temperature`) do typického měsíce 1–12 — **ne** Open-Meteo Climate API (SST nenese).
- Oblast bez marine bodu → `water_temp_*` zůstávají `NULL`; ingest SST se nespouští.

**Trips (cache pro katalog):** `water_temp_min_c` / `water_temp_max_c` + `water_temperature_source` (stejný enum `temperature_source` jako u vzduchu). Autor nevyplňuje ručně. Agregace — viz [Agregace na úrovni `trips`](implementation-notes.md#agregace-na-úrovni-trips).

**Lookup vody** (paralelní větev ke vzduchové teplotě):

1. Stejné kombinace `(place.weather_region_id, week_start)` ze všech typů segmentů jako v [Lookup počasí podle typu segmentu](#lookup-počasí-podle-typu-segmentu).
2. Geo-řetězec: první hit, kde má `weather_records` **vyplněné** `water_temp_*` (ne jen jakýkoli týdenní řádek — rodič bez SST se přeskočí, i když má vzduchová data).
3. Chybí-li SST v celém řetězci → pro trip water cache zkusit `weather_climate_months.water_temp_*` ve stejném řetězci (měsíc středu týdne — stejné pravidlo jako [Fallback na klima](#fallback-na-klima)). První hit s vyplněnou vodou vyhrává.
4. Efektivní min/max: `water_temp_min_c` / `water_temp_max_c`; pokud chybí, fallback na `water_temp_avg_c` pro daný směr. Kombinace bez jakékoli water hodnoty → přeskočit (nepřispívá do trip cache).
5. Agregace: `water_temp_min_c = MIN`, `water_temp_max_c = MAX` přes deduplikované `(resolved_weather_region_id, week_start)` s dostupnou vodou.
6. `water_temperature_source`: `weather_records` když všechny přispívající kombinace z týdenních dat; `climate` když alespoň jedna z klimatu; `NULL` když žádná.

**Klimatický fallback vody platí jen pro trip water cache.** Balení a robotaxi advisories vodu ve v1 **nepoužívají**.

**Katalog:** filtr „voda alespoň X °C“ vyžaduje `water_temp_min_c IS NOT NULL`. Vnitrozemský výlet (Praha) má cache `NULL` a vodním filtrem neprojde — ve veřejném katalogu bez vodního filtru zůstává viditelný.

FE i18n: např. „Teplota moře“; u `water_temperature_source = 'climate'` orientační dlouhodobý průměr pro dané období.


### Počasí a oblasti

Počasí se neváže přímo na jednotlivé místo, ale na **oblast** (`weather_regions`). Místo (`places.weather_region_id`) odkazuje na oblast, ze které se pro plánování výletu načítá týdenní počasí.

| Koncept | Popis |
|---|---|
| **Oblast** | Administrativní nebo custom zóna identifikovaná ISO kódy (`country_code`, `subdivision_*`, `locality`, `postal_code`) nebo `region_type = custom`. Více míst může sdílet jednu oblast. |
| **Typ oblasti** | `postal_code` (PSČ) → `locality` (město) → `subdivision` (stát/kraj) → `custom` (ručně kurátorovaná). V1 primárně ukládá `weather_records` na úrovni `locality`; `postal_code` jen kde klima v rámci města kolísá. Hierarchie bez `parent_id` — viz [geo-fallback](#hierarchie-oblastí-a-geo-fallback). |
| **Časová zóna** | `weather_regions.timezone` je IANA identifikátor a zdroj pravdy pro lokální ISO týden (`Europe/Prague`, `America/Los_Angeles`). |
| **Týdenní záznam** | Jeden řádek v `weather_records` = jeden lokální ISO týden v dané oblasti. |
| **Měsíční klima** | Jeden řádek v `weather_climate_months` = typický kalendářní měsíc (1–12) v dané oblasti — klimatický normál, ne konkrétní rok. |
| **Teplota (vzduchová)** | Průměr + volitelné min/max za týden (nebo typicky za měsíc u klimatu). Kanon °C; FE zobrazení dle `users.unit_system` — viz [Jednotky zobrazení](users-and-trips.md#jednotky-zobrazení). |
| **Vlhkost** | Volitelná průměrná relativní vlhkost `%` (`humidity_avg_pct`) — vstup pro pocitovou teplotu; bez přepočtu jednotek. |
| **Pocitová teplota** | Průměr + volitelné min/max (`feels_like_*`) — *apparent temperature* z teploty, vlhkosti a větru; viz [Pocitová teplota](#pocitová-teplota). Kanon °C; FE dle `unit_system`. |
| **Teplota mořské vody (SST)** | Průměr + volitelné min/max (`water_temp_*`) — povrch moře/oceánu; jen u oblastí s marine bodem; viz [Teplota mořské vody](#teplota-mořské-vody-sst). Kanon °C; FE dle `unit_system`. |
| **Marine bod** | `marine_latitude` / `marine_longitude` na `weather_regions` — souřadnice pro SST ingest (ne centroid města). |
| **Oblačnost / slunce** | `sky_condition` (jasno → zataženo) + volitelně součet hodin slunečního svitu. |
| **Déšť** | Intenzita (`precipitation_intensity`), `rain_mm` (mm vody) a počet deštivých dní (0–7 týden / 0–31 měsíc). Kanon mm; FE dle `unit_system` (mm / in). |
| **Vítr** | Síla (`wind_force`) + u týdenního záznamu volitelně průměrná/max rychlost (m/s) a směr větru. FE rychlost dle `unit_system` (m/s / mph). |
| **Mlha** | Typ (`fog_condition`), počet mlhavých dní a u týdenního záznamu průměrná viditelnost (metry; FE dle `unit_system`). |

#### Týdenní granularita (finální pro v1)

Segmenty v rámci stejného lokálního ISO týdne dle `weather_regions.timezone` sdílí jeden `weather_records` záznam — např. segment v pátek večer a v sobotu dopoledne používají stejný týdenní průměr. Segment protínající hranici týdne v lokálním čase dané oblasti (např. noční ubytovací úsek neděle 22:00 → pondělí 08:00) načítá počasí z **obou** dotčených týdnů. Pro doporučení oblečení a plánování balení to stačí; **denní `weather_records` jsou mimo scope v1**.

**Omezení v1:**

- Víkend s pátečním deštěm a sobotním sluncem sdílí jeden týdenní průměr (oba dny jsou ve stejném lokálním ISO týdnu).
- Cache balení agreguje na úroveň výletu (`trip_packing_items`) — nelze generovat „v pátek deštník, v sobotu opalovák“ bez denních dat; per-segment detail je v `segment_packing_items`.
- Uvnitř jednoho týdne nelze rozlišit denní variace; segment přes hranici týdne **zahrnuje všechny protínající týdny**.

**Backlog v2:** tabulka `weather_records_daily` (region × den).

#### Hierarchie oblastí a geo-fallback

Oblasti jsou **ploché řádky** bez sloupce `parent_weather_region_id`. Hierarchie `postal_code` → `locality` → `subdivision` se odvodí z ISO polí a `region_type` (aplikační logika). `places.weather_region_id` zůstává jeden FK na nejjemnější dostupnou oblast; fallback se děje jen při **čtení** dat, ne při párování místa.

**Odvodění rodiče:**

| Aktuální `region_type` | Najít rodiče |
|---|---|
| `postal_code` | `locality` se stejným `(country_code, subdivision_code, locality)` |
| `locality` | `subdivision` se stejným `(country_code, subdivision_code)` |
| `subdivision` | konec řetězce |
| `custom` | konec řetězce (bez rodiče ve v1) |

Pokud rodičovský řádek v `weather_regions` neexistuje, krok přeskočit. `custom` oblasti nemají geo-fallback ve v1 (není jednoznačný rodič z ISO kódů).

**Timezone:** výpočet lokálních ISO týdnů vždy z timezone **původní** oblasti místa (`places.weather_region_id`), ne z rodiče — jinak by se při rozdílné TZ mohly posunout hranice týdne.

**Subdivision jako fallback:** státní / krajská oblast je hrubší (ingest na `center_*` = centroid, ne agregát měst). Je záměrně záložní zdroj, když chybí data na jemnější úrovni — ne primární volba pro místo ve městě.

**Pořadí zdrojů** pro jednu kombinaci `(place.weather_region_id, week_start)`:

1. `weather_records` na oblasti místa
2. `weather_records` na rodičích v řetězci (první hit)
3. Jen pro teplotní cache na `trips` (vzduchová / pocitová / SST): `weather_climate_months` ve stejném geo-řetězci (viz [Fallback na klima](#fallback-na-klima); u vody jen řádky s vyplněnými `water_temp_*` — viz [Teplota mořské vody](#teplota-mořské-vody-sst))
4. Jinak kombinaci přeskočit

Geo-fallback na rodičovské `weather_records` platí pro **všechny** konzumenty lookupu (teplota, SST, balení, robotaxi, itinerář). U SST je hit platný jen když má záznam vyplněné `water_temp_*`. Klimatický fallback zůstává jen pro teplotní cache na `trips` (vzduchová, pocitová i voda).

**Deduplikace** při agregaci teploty: klíč je `(resolved_weather_region_id, week_start)` — region, ze kterého skutečně přišel záznam (ne nutně `places.weather_region_id`). LA i Santa Monica, které obě spadnou na Kalifornii pro stejný týden, se tak započítají jednou.

#### Lookup počasí podle typu segmentu

Kanonický algoritmus pro teplotu, balení i zobrazení v itineráři:

1. Pro každý segment a vyhodnocovaný region převést interval `[start_time, end_time)` do `weather_regions.timezone` **původní** oblasti místa (`places.weather_region_id`) a určit **všechny lokální ISO týdny** (`week_start`), které interval protíná — každé pondělí dotčeného týdne. U polootevřeného intervalu se `end_time` **nepočítá** — segment končící v pondělí 00:00 lokálního času neprotíná nový ISO týden.
2. Pro každou kombinaci `(place.weather_region_id, week_start)` sestavit kandidáty podle tabulky (start = oblast místa):

| `segment_kind` | Lookup regionů |
|---|---|
| `accommodation` | `start_place_id.weather_region_id` × každý protínající týden |
| `transit` | `start_place_id.weather_region_id`; pokud `end_place_id IS NOT NULL`, i `end_place_id.weather_region_id` — každá kombinace region × protínající týden se vyhodnotí zvlášť (pro oblečení i agregaci teploty) |
| `activity` | `start_place_id.weather_region_id`; pokud `end_place_id IS NOT NULL`, i `end_place_id.weather_region_id` — každá kombinace region × protínající týden se vyhodnotí zvlášť (pro oblečení i agregaci teploty) |

3. Chybí `weather_region_id` → danou kombinaci přeskočit. Jinak projít [geo-řetězec](#hierarchie-oblastí-a-geo-fallback) a načíst první dostupný `weather_records` pro `(candidate_region_id, week_start)`. Hit na rodiči je platný týdenní záznam (balení, robotaxi i teplota). Chybí-li týdenní záznam v celém řetězci → pro weather-based pravidla (balení, robotaxi upozornění) kombinaci přeskočit; pro **agregaci teploty na `trips`** se nejdřív zkusí klimatický normál ve stejném geo-řetězci (viz [Fallback na klima](#fallback-na-klima)) a teprve pak se kombinace přeskočí. Vyřešený region (`resolved_weather_region_id`) použít pro deduplikaci teploty.
4. Z každého načteného záznamu spočítat efektivní teploty pro agregaci na `trips` i pro `clothing_rules`:
   - **Vzduchová:** `temp_min_c` / `temp_max_c`; pokud chybí (`NULL`), fallback na `temp_avg_c` (NOT NULL) pro daný směr.
   - **Pocitová:** `feels_like_min_c` / `feels_like_max_c`; pokud chybí, fallback na `feels_like_avg_c`; pokud chybí i ta, fallback na **efektivní vzduchovou** stejného směru (min → efektivní `temp_min`, max → efektivní `temp_max`, avg → `temp_avg_c`). Pravidlo nikdy nevyhodnocuje teplotní podmínku proti `NULL`.
   - **Mořská voda (SST):** samostatná větev — viz [Teplota mořské vody](#teplota-mořské-vody-sst). Hit pro vzduch neznamená automaticky hit pro vodu; rodič bez `water_temp_*` se pro SST přeskočí. Efektivní min/max: `water_temp_*` s fallbackem na `water_temp_avg_c`.
   - Podmínky `clothing_rules.temp_*` matchují efektivní vzduchovou; podmínky `clothing_rules.feels_like_*` matchují efektivní pocitovou (viz priorita v [clothing_rules](packing.md#clothing_rules)). Voda se v `clothing_rules` ve v1 nepoužívá.

#### Pocitová teplota

Pocitová teplota vyjadřuje, jak teplota působí na člověka s ohledem na **vlhkost** a **vítr** — ne jen údaj teploměru.

**Definice (v1):** **pocitová teplota** (*apparent temperature*) ve °C — jak teplota působí na člověka s ohledem na vlhkost a vítr. Preferovaný zdroj hodnot: Open-Meteo pole `apparent_temperature`. Vstupy modelu: vzduchová teplota, relativní vlhkost, rychlost větru.

**Chování v kostce** (pro produkt / FE):

- V **horku** vysoká vlhkost pocitovou teplotu **zvyšuje** (brání odpařování potu — dusno).
- V **chladu** vítr pocitovou teplotu **snižuje** (wind chill); vliv vlhkosti je v běžném *apparent temperature* modelu slabší než u plné **vlezlé zimy** (*damp cold*), ale model ji zohledňuje.
- V suchém horku / bezvětří je pocitová blízká vzduchové.

**Kde se ukládá:** `weather_records` a `weather_climate_months` — sloupce `humidity_avg_pct`, `feels_like_avg_c`, `feels_like_min_c`, `feels_like_max_c`. PostgreSQL **nevypočítává** vzorec; ukládá hodnoty z ingestu / aplikační vrstvy.

**Ingest (preferovaný):**

- Týdenní / forecast: Open-Meteo (nebo ekvivalent) — mapovat `apparent_temperature` → `feels_like_*`, `relative_humidity_2m` → `humidity_avg_pct`, plus stávající `temp_*` a vítr.
- Pokud API dodá jen vstupy bez `apparent_temperature`, aplikace spočítá pocitovou teplotu (*apparent temperature*) z `temp_*`, `humidity_avg_pct` a `wind_speed_avg_ms` a uloží výsledek.
- Klima (`weather_climate_months`): pokud Climate API nedává `apparent_temperature`, job spočítá pocitovou teplotu z typické teploty, vlhkosti a odhadu větru — `wind_force` → střed orientačního pásma m/s z tabulky [`wind_force`](#wind_force) (např. `moderate` → ~7,5 m/s).

**Použití ve v1:**

| Místo | Role |
|---|---|
| `trips.feels_like_min_c` / `feels_like_max_c` | Primární katalogový filtr „teplota“ |
| `trips.temp_min_c` / `temp_max_c` | Vzduchová cache — zobrazení / sekundární filtr |
| `trips.water_temp_min_c` / `water_temp_max_c` | SST cache — zobrazení / katalogový filtr „teplota vody“ |
| Skóre „Kdy jet“ | Teplotní pásmo a prahy na pocitové (fallback vzduchová); SST skóre neovlivňuje |
| `clothing_rules` | Seed v1 na `temp_*`; volitelné `feels_like_*` prahy mají prioritu, pokud jsou na rule vyplněné |

**Mimo scope v1:** vlastní korekce vlezlé zimy (*damp cold*) nad běžným *apparent temperature*; denní granularita pocitové (`weather_records_daily`).

#### Přiřazení místa k oblasti (hybrid)

Auto-párování volí **nejjemnější** dostupnou oblast a uloží ji do `places.weather_region_id`. Runtime [geo-fallback](#hierarchie-oblastí-a-geo-fallback) je oddělená vrstva — při lookupu počasí, ne při přiřazení místa.

1. Exact match: `places.country_code` + `places.postal_code` → `weather_regions` s `region_type = 'postal_code'`.
2. Město: `country_code` + `subdivision_code` + `locality` → `region_type = 'locality'`.
3. Hrubý fallback: `country_code` + `subdivision_code` → `region_type = 'subdivision'`.
4. Poslední fallback: nejbližší `center_latitude` / `center_longitude` mezi regiony **stejné země** (ne globálně).
5. `region_type = 'custom'` se pro auto-návrh nepoužívá — vyžaduje ruční přiřazení.
6. Admin v UI návrh potvrdí nebo přiřadí jinou oblast ručně (`places.weather_region_id`), ale aplikace stále vynucuje shodu země a kontroluje dostupná jemnější geo pole podle typu regionu.

#### Chybějící `weather_region_id`

Pokud `places.weather_region_id IS NULL`, weather-based podmínky v `clothing_rules` (teplotní sloupce, junction tabulky pro počasí) se pro daný segment přeskočí. Segment-only podmínky (`clothing_rule_difficulties`, `non_accommodation_activity`, `min_duration_minutes_gte`) se vyhodnotí normálně.

#### `sky_condition`

Referenční popisy níže patří do FE locale souborů, ne do PostgreSQL. Pořadí deklarace enumu odpovídá **rostoucí zataženosti** — `MAX()` tedy vrací nejzataženější hodnotu. `variable` je záměrně uprostřed škály: proměnlivý týden není horší než souvisle zatažený, takže nesmí v agregaci přebít `overcast`.

| Hodnota | FE i18n (příklad) |
|---|---|
| `clear` | Jasno — bez oblačnosti |
| `mostly_sunny` | Převážně slunečno |
| `partly_cloudy` | Polojasno — střídání slunce a mraků |
| `variable` | Proměnlivé oblačno během týdne |
| `mostly_cloudy` | Převážně oblačno |
| `cloudy` | Oblačno |
| `overcast` | Zataženo — souvislá oblačnost |

#### `wind_force`

Referenční popisy níže patří do FE locale souborů, ne do PostgreSQL.

| Hodnota | FE i18n (příklad) | Orientačně (m/s) |
|---|---|---|
| `calm` | Bezvětří | 0–1 |
| `light` | Slabý vítr | 1–5 |
| `moderate` | Mírný vítr | 5–10 |
| `fresh` | Čerstvý vítr | 10–15 |
| `strong` | Silný vítr | 15–20 |
| `gale` | Větrno / bouřlivý vítr | 20–30 |
| `storm` | Vichřice | 30+ |

#### `fog_condition`

Referenční popisy níže patří do FE locale souborů, ne do PostgreSQL.

| Hodnota | FE i18n (příklad) |
|---|---|
| `none` | Bez mlhy |
| `haze` | Opar / lehká zamlženost |
| `mist` | Mlha — viditelnost obvykle 1–2 km |
| `fog` | Hustá mlha — viditelnost pod ~1 km |
| `dense_fog` | Velmi hustá mlha — viditelnost pod ~200 m |

Typický dotaz pro itinerář: pro každý segment určit všechny protínající lokální ISO týdny, načíst `weather_records` podle pravidel v [Lookup počasí podle typu segmentu](#lookup-počasí-podle-typu-segmentu).

Data `weather_records` lze plnit z externího API (Open-Meteo, Visual Crossing apod.) a ukládat jako historický archiv i prognózu pro blízké týdny. Preferované mapování pocitové teploty a vlhkosti: viz [Pocitová teplota](#pocitová-teplota).

#### Ideální měsíce (klima)

**Zdroj pravdy:** `weather_climate_months` (region × kalendářní měsíc 1–12) — ne `weather_records`, ne segment, ne trip.

Účel: na detailu výletu (UI blok „Kdy jet“) ukázat přehled typického počasí a systémové vhodnosti měsíců. Konkrétní termín výletu a balení dál řeší týdenní `weather_records`.

Druhé použití: klima je **fallback pro teplotní cache** na `trips`, když pro dotčené týdny neexistuje týdenní záznam ani na oblasti místa, ani na rodičích v [geo-řetězci](#hierarchie-oblastí-a-geo-fallback) — viz [Fallback na klima](#fallback-na-klima). Skóre ani `suitability` se tím neperzistují, přenáší se jen teplotní metriky.

##### Plnění dat

- Job / admin sync per `weather_regions` z Climate / Historical API (Open-Meteo Climate API nebo ekvivalent) pro vzduchová data.
- Upsert všech 12 měsíců oblasti včetně `humidity_avg_pct` a `feels_like_*` (viz [Pocitová teplota](#pocitová-teplota)); po sync aktualizuj `updated_at`.
- SST (`water_temp_*`): samostatný sync z Marine / Historical archivu na `marine_*` souřadnicích — jen u oblastí s marine bodem; Climate API se pro vodu nepoužívá.
- Chybí-li klima pro oblast, týdenní `weather_records` (balení, `trips.temp_*`, `trips.feels_like_*`, `trips.water_temp_*`) fungují dál — UI jen skryje blok „Kdy jet“, pokud výlet nemá žádné použitelné klimatické řádky.

##### Skóre a labely (aplikace, ne DB)

Vstup: jeden řádek klimatu (regionální normál nebo agregát výletu — viz níže). Skóre a `suitability` se **neperzistují** v PostgreSQL.

Teplotní penalizace používají **pocitovou** teplotu (fallback na vzduchovou, pokud `feels_like_*` chybí). Vlhkost se nepenalizuje zvlášť — už je v pocitové. Samostatná penalizace silného větru zůstává (komfort chůze / robotaxi mimo čistou teplotu).

**Skóre 0–100** — start `100`, odečti penalizace, výsledek clamp na `[0, 100]`:

| Podmínka | Penalizace |
|---|---|
| efektivní `feels_like_avg_c` (fallback `temp_avg_c`) mimo pásmo 16–26 °C | 4 body za každý 1 °C mimo pásmo |
| efektivní pocitové min (`feels_like_min_c` → `feels_like_avg_c` → `temp_min_c` → `temp_avg_c`) &lt; 8 °C | +15 |
| efektivní pocitové max (`feels_like_max_c` → `feels_like_avg_c` → `temp_max_c` → `temp_avg_c`) &gt; 32 °C | +10 |
| `rainy_days` | +2 za každý den nad 6 |
| `rain_mm` | +1 za každých celých 10 mm nad 40 (floor) |
| `precipitation_intensity` ∈ (`heavy`, `storm`) | +20 |
| `sky_condition` ∈ (`overcast`, `cloudy`) | +8 |
| `sunshine_hours` &lt; 120 (jen pokud `NOT NULL`) | +10 |
| `fog_days` &gt; 5 | +8 |
| `wind_force` ∈ (`strong`, `gale`, `storm`) | +12 |

**Label z skóre** (FE i18n ze slugů; DB neukládá):

| Skóre | `suitability` slug |
|---|---|
| ≥ 80 | `ideal` |
| 60–79 | `good` |
| 40–59 | `fair` |
| &lt; 40 | `poor` |

##### Agregace na výlet (read-time)

Na `trips` **není** cache klimatu ani skóre — spočítá se při čtení detailu (12 měsíců × malý počet unikátních regionů).

1. Sesbírat **unikátní** `weather_region_id` z míst segmentů výletu podle stejných pravidel regionů jako v [Lookup počasí podle typu segmentu](#lookup-počasí-podle-typu-segmentu) (`accommodation` → start; `transit` / `activity` → start a případně `end_place_id`). Regiony bez `weather_region_id` přeskočit. Prázdná množina regionů → prázdné `months`, UI skryje blok „Kdy jet“.
2. Pro každý měsíc `1..12`:
   1. Načíst `weather_climate_months` pro všechny unikátní regiony výletu a daný měsíc.
   2. Chybí-li řádek pro **jakýkoli** z těchto regionů → měsíc je nekompletní: v odpovědi vrať `month` s `score = null` a `suitability = null` (metriky `null` nebo částečné z dostupných regionů; FE měsíc nezvýrazní jako vyhodnocený) a **přeskoč** kroky 3–4.
   3. Agregace metrik ze **všech** regionů (konzervativní — nejhorší očekávání pro cestovatele):
      - `temp_min_c` = `MIN` (NULL ignorovat; pokud všechny NULL → NULL)
      - `temp_max_c` = `MAX` (NULL ignorovat)
      - `temp_avg_c` = `AVG` (NOT NULL na řádcích)
      - `feels_like_min_c` = `MIN` (NULL ignorovat)
      - `feels_like_max_c` = `MAX` (NULL ignorovat)
      - `feels_like_avg_c` = `AVG` (NULL ignorovat; pokud všechny NULL → NULL)
      - `water_temp_min_c` = `MIN` (NULL ignorovat; pokud všechny NULL → NULL)
      - `water_temp_max_c` = `MAX` (NULL ignorovat)
      - `water_temp_avg_c` = `AVG` (NULL ignorovat; pokud všechny NULL → NULL)
      - `humidity_avg_pct` = `AVG` (NULL ignorovat; pokud všechny NULL → NULL)
      - `rain_mm` = `MAX`, `rainy_days` = `MAX`, `fog_days` = `MAX`
      - `sunshine_hours` = `MIN` (NULL ignorovat; pokud všechny NULL → NULL)
      - `precipitation_intensity`, `wind_force`, `fog_condition`, `sky_condition` = hodnota s **nejvyšším pořadím v deklaraci enumu** (stejná filozofie jako `MAX` u `segment_difficulty` — pozdější hodnota = horší / intenzivnější). U `sky_condition` je pořadí **rostoucí zataženost** (`clear` … `overcast`) a `variable` leží uprostřed škály mezi `partly_cloudy` a `mostly_cloudy` — jinak by proměnlivý region přebil zatažený a měsíc by unikl penalizaci za `overcast` / `cloudy`.
   4. Na agregovaném měsíčním řádku spočítat skóre a `suitability` dle tabulky výše (teplotní pásmo na pocitové s fallbackem na vzduchovou). **SST do skóre ve v1 nevstupuje** — je informativní metrika, aby se neměnilo chování vnitrozemských výletů.
3. Chybí-li klima pro všechny regiony výletu ve všech měsících → prázdné `months` / UI skryje blok „Kdy jet“.

**API kontrakt** (odpověď detailu výletu — ne tabulka v DB):

```json
{
  "months": [
    {
      "month": 5,
      "temp_avg_c": 18.5,
      "temp_min_c": 12.0,
      "temp_max_c": 24.0,
      "humidity_avg_pct": 62.0,
      "feels_like_avg_c": 19.0,
      "feels_like_min_c": 11.5,
      "feels_like_max_c": 25.5,
      "water_temp_avg_c": 17.0,
      "water_temp_min_c": 15.5,
      "water_temp_max_c": 18.5,
      "rain_mm": 45,
      "rainy_days": 8,
      "score": 82,
      "suitability": "ideal"
    }
  ]
}
```

U vnitrozemského výletu jsou `water_temp_*` v měsíční odpovědi `null` — FE blok teploty moře skryje.

##### Mimo scope v1

- Preference uživatele (**vlastní teplotní pásmo** pro packing / skóre — ne display jednotky; zobrazení °C/°F řídí `users.unit_system`, viz [Jednotky zobrazení](users-and-trips.md#jednotky-zobrazení))
- Cache **skóre / `suitability`** na `trips` a katalogový filtr „výlety ideální v květnu“ (teplotní cache z klimatu je něco jiného — viz [Fallback na klima](#fallback-na-klima))
- Denní klimatická granularita (souvisí s backlogem `weather_records_daily`)
- Ruční override ideálních měsíců autorem výletu
- Vlastní korekce vlezlé zimy (*damp cold*) nad běžným *apparent temperature*
- Teplota vody jezer / bazénů (jen SST moře/oceánu)
- Penalizace / bonus SST ve skóre „Kdy jet“
- `clothing_rules` / robotaxi advisories podle teploty vody

