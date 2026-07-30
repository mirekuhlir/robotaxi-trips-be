# Constraints & indexes

[← Přehled schématu](README.md)

## DB constrainty

DBML soubor je diagramový zdroj pro dbdiagram.io. Některé PostgreSQL vlastnosti (`CHECK`, partial indexy, `ON DELETE` akce, exclusion constrainty) jsou v DBML vyjádřené jen poznámkou; skutečné SQL migrace je musí vytvořit explicitně podle této sekce.

### CHECK constrainty (PostgreSQL)

**`segments`:**

```sql
CHECK (end_time > start_time)
CHECK (local_price_amount >= 0)
CHECK (home_price_amount >= 0)
CHECK (price_usd >= 0)
CHECK (exchange_rate_local_to_usd > 0)
CHECK (exchange_rate_local_to_home > 0)
CHECK (local_price_currency ~ '^[A-Z]{3}$')
CHECK (recommended_age_min IS NULL OR recommended_age_max IS NULL OR recommended_age_min <= recommended_age_max)
CHECK (recommended_age_min IS NULL OR recommended_age_min BETWEEN 0 AND 120)
CHECK (recommended_age_max IS NULL OR recommended_age_max BETWEEN 0 AND 120)
```

**`trips`:**

```sql
CHECK (total_cost_usd >= 0)
CHECK (total_cost_home >= 0)
CHECK (home_currency ~ '^[A-Z]{3}$')
CHECK (total_duration_minutes >= 0)
CHECK (recommended_age_min IS NULL OR recommended_age_max IS NULL OR recommended_age_min <= recommended_age_max)
CHECK (recommended_age_min IS NULL OR recommended_age_min BETWEEN 0 AND 120)
CHECK (recommended_age_max IS NULL OR recommended_age_max BETWEEN 0 AND 120)
CHECK (temp_min_c IS NULL OR temp_max_c IS NULL OR temp_min_c <= temp_max_c)
CHECK (temp_min_c IS NULL OR temp_min_c BETWEEN -90 AND 60)
CHECK (temp_max_c IS NULL OR temp_max_c BETWEEN -90 AND 60)
CHECK (feels_like_min_c IS NULL OR feels_like_max_c IS NULL OR feels_like_min_c <= feels_like_max_c)
CHECK (feels_like_min_c IS NULL OR feels_like_min_c BETWEEN -90 AND 60)
CHECK (feels_like_max_c IS NULL OR feels_like_max_c BETWEEN -90 AND 60)
CHECK (rating_avg IS NULL OR (rating_avg >= 1 AND rating_avg <= 5))
CHECK (rating_count >= 0)
CHECK (
  (rating_count = 0 AND rating_avg IS NULL)
  OR (rating_count > 0 AND rating_avg IS NOT NULL)
)
```

**`users`:**

```sql
CHECK (home_currency ~ '^[A-Z]{3}$')
```

**`weather_regions`:**

```sql
CHECK (country_code ~ '^[A-Z]{2}$')
CHECK (region_type <> 'postal_code' OR postal_code IS NOT NULL)
CHECK (region_type <> 'locality' OR locality IS NOT NULL)
CHECK (region_type <> 'locality' OR subdivision_code IS NOT NULL)
CHECK (region_type <> 'locality' OR postal_code IS NULL)
CHECK (region_type <> 'subdivision' OR subdivision_code IS NOT NULL)
CHECK (region_type <> 'subdivision' OR (locality IS NULL AND postal_code IS NULL))
CHECK (center_latitude BETWEEN -90 AND 90)
CHECK (center_longitude BETWEEN -180 AND 180)
CHECK (timezone <> '')
```

**`places`:**

```sql
CHECK (country_code IS NULL OR country_code ~ '^[A-Z]{2}$')
CHECK (latitude BETWEEN -90 AND 90)
CHECK (longitude BETWEEN -180 AND 180)
CHECK (rating IS NULL OR (rating >= 1 AND rating <= 5))
CHECK (review_rating_avg IS NULL OR (review_rating_avg >= 1 AND review_rating_avg <= 5))
CHECK (review_rating_count >= 0)
CHECK (
  (review_rating_count = 0 AND review_rating_avg IS NULL)
  OR (review_rating_count > 0 AND review_rating_avg IS NOT NULL)
)
CHECK (phone_calling_code IS NULL OR phone_calling_code ~ '^[0-9]{1,3}$')
```

Neznámá / nezadaná platba = `accepted_payments IS NULL`. Žádný CHECK ani index navíc — hodnoty pokrývá typ `place_accepted_payments`.

**`trip_reviews` / `place_reviews`:**

```sql
CHECK (score BETWEEN 1 AND 5)
```

**`trip_review_media` / `place_review_media`:**

```sql
CHECK (sort_order >= 0)
CHECK (
  (media_kind = 'image' AND poster_url IS NULL)
  OR media_kind = 'video'
)
```

**`weather_records`:**

```sql
CHECK (EXTRACT(ISODOW FROM week_start) = 1)
CHECK (temp_avg_c BETWEEN -90 AND 60)
CHECK (temp_min_c IS NULL OR temp_min_c BETWEEN -90 AND 60)
CHECK (temp_max_c IS NULL OR temp_max_c BETWEEN -90 AND 60)
CHECK (temp_min_c IS NULL OR temp_max_c IS NULL OR temp_min_c <= temp_max_c)
CHECK (
  temp_min_c IS NULL OR temp_max_c IS NULL
  OR (temp_avg_c >= temp_min_c AND temp_avg_c <= temp_max_c)
)
CHECK (humidity_avg_pct IS NULL OR (humidity_avg_pct >= 0 AND humidity_avg_pct <= 100))
CHECK (feels_like_avg_c IS NULL OR feels_like_avg_c BETWEEN -90 AND 60)
CHECK (feels_like_min_c IS NULL OR feels_like_min_c BETWEEN -90 AND 60)
CHECK (feels_like_max_c IS NULL OR feels_like_max_c BETWEEN -90 AND 60)
CHECK (feels_like_min_c IS NULL OR feels_like_max_c IS NULL OR feels_like_min_c <= feels_like_max_c)
CHECK (
  feels_like_avg_c IS NULL OR feels_like_min_c IS NULL OR feels_like_max_c IS NULL
  OR (feels_like_avg_c >= feels_like_min_c AND feels_like_avg_c <= feels_like_max_c)
)
CHECK (sunshine_hours IS NULL OR (sunshine_hours >= 0 AND sunshine_hours <= 168))
CHECK (rain_mm >= 0)
CHECK (rainy_days BETWEEN 0 AND 7)
CHECK (fog_days BETWEEN 0 AND 7)
CHECK (wind_speed_avg_ms IS NULL OR wind_speed_avg_ms >= 0)
CHECK (wind_speed_max_ms IS NULL OR wind_speed_max_ms >= 0)
CHECK (wind_speed_avg_ms IS NULL OR wind_speed_max_ms IS NULL OR wind_speed_avg_ms <= wind_speed_max_ms)
CHECK (wind_direction_avg_deg IS NULL OR wind_direction_avg_deg BETWEEN 0 AND 359)
CHECK (visibility_avg_m IS NULL OR visibility_avg_m >= 0)
CHECK (precipitation_intensity <> 'none' OR (rain_mm = 0 AND rainy_days = 0))
CHECK (fog_condition <> 'none' OR fog_days = 0)
```

**`weather_climate_months`:**

```sql
CHECK (month BETWEEN 1 AND 12)
CHECK (temp_avg_c BETWEEN -90 AND 60)
CHECK (temp_min_c IS NULL OR temp_min_c BETWEEN -90 AND 60)
CHECK (temp_max_c IS NULL OR temp_max_c BETWEEN -90 AND 60)
CHECK (temp_min_c IS NULL OR temp_max_c IS NULL OR temp_min_c <= temp_max_c)
CHECK (
  temp_min_c IS NULL OR temp_max_c IS NULL
  OR (temp_avg_c >= temp_min_c AND temp_avg_c <= temp_max_c)
)
CHECK (humidity_avg_pct IS NULL OR (humidity_avg_pct >= 0 AND humidity_avg_pct <= 100))
CHECK (feels_like_avg_c IS NULL OR feels_like_avg_c BETWEEN -90 AND 60)
CHECK (feels_like_min_c IS NULL OR feels_like_min_c BETWEEN -90 AND 60)
CHECK (feels_like_max_c IS NULL OR feels_like_max_c BETWEEN -90 AND 60)
CHECK (feels_like_min_c IS NULL OR feels_like_max_c IS NULL OR feels_like_min_c <= feels_like_max_c)
CHECK (
  feels_like_avg_c IS NULL OR feels_like_min_c IS NULL OR feels_like_max_c IS NULL
  OR (feels_like_avg_c >= feels_like_min_c AND feels_like_avg_c <= feels_like_max_c)
)
CHECK (sunshine_hours IS NULL OR (sunshine_hours >= 0 AND sunshine_hours <= 744))
CHECK (rain_mm >= 0)
CHECK (rainy_days BETWEEN 0 AND 31)
CHECK (fog_days BETWEEN 0 AND 31)
CHECK (precipitation_intensity <> 'none' OR (rain_mm = 0 AND rainy_days = 0))
CHECK (fog_condition <> 'none' OR fog_days = 0)
```

**`clothing_rules`:**

```sql
CHECK (temp_avg_c_lte IS NULL OR temp_avg_c_lte BETWEEN -90 AND 60)
CHECK (temp_avg_c_gte IS NULL OR temp_avg_c_gte BETWEEN -90 AND 60)
CHECK (temp_min_c_lte IS NULL OR temp_min_c_lte BETWEEN -90 AND 60)
CHECK (temp_min_c_gte IS NULL OR temp_min_c_gte BETWEEN -90 AND 60)
CHECK (temp_max_c_lte IS NULL OR temp_max_c_lte BETWEEN -90 AND 60)
CHECK (temp_max_c_gte IS NULL OR temp_max_c_gte BETWEEN -90 AND 60)
CHECK (temp_avg_c_gte IS NULL OR temp_avg_c_lte IS NULL OR temp_avg_c_gte <= temp_avg_c_lte)
CHECK (temp_min_c_gte IS NULL OR temp_min_c_lte IS NULL OR temp_min_c_gte <= temp_min_c_lte)
CHECK (temp_max_c_gte IS NULL OR temp_max_c_lte IS NULL OR temp_max_c_gte <= temp_max_c_lte)
CHECK (feels_like_avg_c_lte IS NULL OR feels_like_avg_c_lte BETWEEN -90 AND 60)
CHECK (feels_like_avg_c_gte IS NULL OR feels_like_avg_c_gte BETWEEN -90 AND 60)
CHECK (feels_like_min_c_lte IS NULL OR feels_like_min_c_lte BETWEEN -90 AND 60)
CHECK (feels_like_min_c_gte IS NULL OR feels_like_min_c_gte BETWEEN -90 AND 60)
CHECK (feels_like_max_c_lte IS NULL OR feels_like_max_c_lte BETWEEN -90 AND 60)
CHECK (feels_like_max_c_gte IS NULL OR feels_like_max_c_gte BETWEEN -90 AND 60)
CHECK (feels_like_avg_c_gte IS NULL OR feels_like_avg_c_lte IS NULL OR feels_like_avg_c_gte <= feels_like_avg_c_lte)
CHECK (feels_like_min_c_gte IS NULL OR feels_like_min_c_lte IS NULL OR feels_like_min_c_gte <= feels_like_min_c_lte)
CHECK (feels_like_max_c_gte IS NULL OR feels_like_max_c_lte IS NULL OR feels_like_max_c_gte <= feels_like_max_c_lte)
CHECK (rainy_days_gte IS NULL OR rainy_days_gte BETWEEN 0 AND 7)
CHECK (min_duration_minutes_gte IS NULL OR min_duration_minutes_gte >= 0)
```

**`travel_requirement_rules`:**

```sql
CHECK (min_distinct_countries_gte IS NULL OR min_distinct_countries_gte >= 1)
```

**`segment_images`:**

```sql
CHECK (sort_order >= 0)
```

**`clothing_items` / `travel_requirement_items`:**

```sql
CHECK (sort_order >= 0)
```

**`transit_details`:**

```sql
CHECK (distance_meters IS NULL OR distance_meters >= 0)
CHECK (estimated_wait_minutes IS NULL OR estimated_wait_minutes >= 0)
CHECK (passenger_count IS NULL OR passenger_count >= 1)
```

**`robotaxi_vehicle_models`:**

```sql
CHECK (seat_count >= 1)
```

**`provider_service_areas`:**

```sql
CHECK (country_code ~ '^[A-Z]{2}$')
CHECK (operates_24_7 = true OR (daily_start_local IS NOT NULL AND daily_end_local IS NOT NULL))
```

**`robotaxi_advisory_rules`:**

```sql
CHECK (visibility_avg_m_lte IS NULL OR visibility_avg_m_lte >= 0)
```

**`robotaxi_advisory_items`:**

```sql
CHECK (sort_order >= 0)
```

### Validace v aplikační vrstvě

Tyto invarianty PostgreSQL CHECK neřeší — vynucuj je aplikace při zápisu:

- `users.home_currency` a `trips.home_currency` musí být validní ISO 4217 uppercase (`^[A-Z]{3}$`)
- změna `trips.home_currency` vyžaduje v jedné transakci přepočet všech segmentů výletu (`home_price_amount`, `exchange_rate_local_to_home`) a `total_cost_home`
- u `segment_kind = accommodation` musí split ceny držet konzistenci napříč vrstvami: `SUM(local_price_amount)`, `SUM(home_price_amount)` a `SUM(price_usd)` odpovídají celkové ceně rezervace
- segmenty jednoho výletu se modelují jako **polootevřené intervaly** `[start_time, end_time)`; produkční DB vynucuje nepřekrývání exclusion constraintem níže. Aplikace stále validuje před zápisem kvůli lepší chybové hlášce; dotyk na hranici je povolen (`end_time` segmentu A = `start_time` segmentu B není překryv); mezery mezi segmenty jsou povolené
- při INSERT/UPDATE/DELETE segmentů v aplikační vrstvě používej jednu transakci se zámkem na výlet (`SELECT * FROM trips WHERE id = :trip_id FOR UPDATE`) nebo advisory lockem per `trip_id`; DB constraint je poslední ochrana proti souběžným zápisům
- u segmentů se stejným `trip_id` nesmí dva segmenty s `end_time > start_time` sdílet stejný `start_time` — timeline je striktně lineární, bez paralelních větví; pravidlo je redundantní s nepřekrýváním, ale explicitně garantuje deterministické `ORDER BY start_time, id` bez paralelních větví
- `transit_details` existuje jen u segmentů s `segment_kind = transit`
- konzistence `segment_kind` ↔ `end_place_id` / `transit_details` / kategorie místa / `recommended_age_*` / `difficulty` (viz [Sémantika segmentů](segments.md#sémantika-segmentů))
- `recommended_age_min` / `recommended_age_max` jen u `segment_kind = activity`; u `accommodation` a `transit` vždy NULL
- `difficulty` u `accommodation` vždy NULL; u `activity` volitelné (agregace `max_difficulty`); u `transit` volitelné (jen `clothing_rules`, ne agregace)
- na každém výletu musí být v `trip_members` alespoň jeden admin — validace při DELETE člena nebo UPDATE role
- před `DELETE FROM users` ověřit, že uživatel není **jediný admin** na žádném výletu — jinak chyba validace (CASCADE na `trip_members.user_id` by jinak porušil invariant admina; RESTRICT na `created_by` to neřeší, tvůrce může být už odstraněn z `trip_members`)
- agregace věku aktivit nesmí vést k `recommended_age_min > recommended_age_max` na `trips` — jinak poruší DB CHECK; aplikace detekuje konflikt při uložení segmentu před COMMIT
- `places.weather_region_id` smí odkazovat jen na `weather_regions` ve stejné zemi (`places.country_code = weather_regions.country_code`, pokud je `places.country_code` vyplněné); při ručním přiřazení validuj i jemnější shodu podle typu regionu (`postal_code`, `locality`, `subdivision`), pokud jsou příslušná pole na místě známá
- u `places` prázdné řetězce `name` / `description` / `website_url` / `address` / `phone_calling_code` / `telephone` normalizuj na `NULL`; `website_url` při vyplnění musí být HTTPS URL; `phone_calling_code` jen číslice bez vedoucího `+` (1–3 znaky); `telephone` bez předvolby — FE pro `tel:` odkaz spojí číslice z obou polí; externí `rating` (Google Maps–style) volitelné, při vyplnění `1.0`–`5.0` — **ne** přepisovat z `place_reviews`
- pravidla zápisu a agregace uživatelských recenzí (`trip_reviews`, `place_reviews`, media) — viz [Recenze](users-and-trips.md#recenze)
- `clothing_rules` musí mít alespoň jednu aktivní podmínku (skalární sloupec, `non_accommodation_activity = true`, nebo ≥1 řádek v junction tabulce) — pravidlo bez podmínky je neplatné
- `travel_requirement_rules` musí mít alespoň jednu aktivní podmínku (skalární sloupec NOT NULL) — pravidlo bez podmínky je neplatné
- `robotaxi_advisory_rules` musí mít alespoň jednu aktivní podmínku (skalární sloupec NOT NULL nebo ≥1 řádek v junction tabulce) — pravidlo bez podmínky je neplatné
- u `transit_details` s `transport_mode = robotaxi`: `provider_id` NOT NULL; `pickup_zone_place_id`, `dropoff_zone_place_id`, `estimated_wait_minutes`, `passenger_count`, `vehicle_model_id` smí být vyplněné jen u robotaxi — u ostatních režimů vždy `NULL`
- u `transit_details` s `transport_mode <> robotaxi`: `provider_id`, `pickup_zone_place_id`, `dropoff_zone_place_id`, `estimated_wait_minutes`, `passenger_count`, `vehicle_model_id` vždy `NULL`
- `pickup_zone_place_id` / `dropoff_zone_place_id` musí odkazovat na `places` s kategorií `robotaxi_pickup_zone`
- `vehicle_model_id` musí odkazovat na model se stejným `provider_id` jako `transit_details.provider_id`
- `passenger_count` nesmí překročit `robotaxi_vehicle_models.seat_count`, pokud je `vehicle_model_id` vyplněné

Produkční minimum pro nepřekrývání segmentů je DB-level exclusion constraint přes `btree_gist`:

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE segments
  ADD CONSTRAINT segments_no_overlap
  EXCLUDE USING gist (
    trip_id WITH =,
    tstzrange(start_time, end_time, '[)') WITH &&
  );
```

---

## Přehled indexů

| Tabulka | Index | Účel |
|---|---|---|
| `users` | `idx_users_created_at` | Řazení / paginace uživatelů |
| `trips` | `idx_trips_created_by` | Audit / admin dotazy na tvůrce výletu (ne „Moje výlety" — ten jde přes `trip_members`) |
| `trips` | `idx_trips_visibility` | Filtrování dle viditelnosti |
| `trips` | `idx_trips_status_visibility` | Kombinovaný filtr pro katalog; pokrývá i samostatný filtr dle `status` (leading sloupec) |
| `trips` | `idx_trips_total_cost_usd` | Řazení a filtrování dle ceny ve veřejném katalogu (USD kanon) |
| `trips` | `idx_trips_total_cost_home` | Řazení „mých výletů“ v plánovací měně (`home_currency`); **ne** pro veřejný katalog |
| `trips` | `idx_trips_total_duration_minutes` | Řazení a filtrování katalogu dle délky výletu |
| `trips` | `idx_trips_recommended_age` (`recommended_age_min`, `recommended_age_max`) | Filtrování katalogu dle věku cílové skupiny |
| `trips` | `idx_trips_max_difficulty` | Filtrování a řazení katalogu dle náročnosti výletu |
| `trips` | `idx_trips_temp` (`temp_min_c`, `temp_max_c`) | Filtrování katalogu dle vzduchové teploty (sekundární) |
| `trips` | `idx_trips_feels_like` (`feels_like_min_c`, `feels_like_max_c`) | Primární filtrování katalogu dle pocitové teploty |
| `trips` | `idx_trips_rating_avg` | Řazení a filtrování katalogu dle uživatelského hodnocení |
| `trip_members` | `idx_trip_members_user_id` | „Moje výlety" pro daného uživatele |
| `trip_reviews` | `uq_trip_reviews_trip_user` (`trip_id`, `user_id`) | Jeden hlas na uživatele a výlet |
| `trip_reviews` | `idx_trip_reviews_user_id` | „Moje recenze výletů“ |
| `trip_reviews` | `idx_trip_reviews_trip_created_at` (`trip_id`, `created_at`) | Seznam recenzí na detailu výletu |
| `trip_review_media` | `idx_trip_review_media_review_sort` (`review_id`, `sort_order`) | Galerie médií recenze v pořadí `ORDER BY sort_order, id` |
| `places` | `idx_places_coordinates` | Geografické dotazy (blízkost bodu) |
| `places` | `idx_places_review_rating_avg` | Řazení a filtrování dle uživatelského hodnocení místa |
| `places` | `idx_places_weather_region_id` | Místa v dané oblasti počasí |
| `places` | `idx_places_country_code` (partial, `WHERE country_code IS NOT NULL`) | Filtrování / geo pravidla cestovních požadavků |
| `places` | `idx_places_postal_code` (partial, `WHERE country_code IS NOT NULL AND postal_code IS NOT NULL`) | Auto-párování místa s `weather_regions` dle PSČ |
| `place_reviews` | `uq_place_reviews_place_user` (`place_id`, `user_id`) | Jeden hlas na uživatele a místo |
| `place_reviews` | `idx_place_reviews_user_id` | „Moje recenze míst“ |
| `place_reviews` | `idx_place_reviews_place_created_at` (`place_id`, `created_at`) | Seznam recenzí na detailu místa |
| `place_review_media` | `idx_place_review_media_review_sort` (`review_id`, `sort_order`) | Galerie médií recenze v pořadí `ORDER BY sort_order, id` |
| `weather_regions` | `uq_weather_regions_postal` (partial, `WHERE region_type = 'postal_code'`) | Unikátní `(country_code, postal_code)` |
| `weather_regions` | `uq_weather_regions_locality` (partial, `WHERE region_type = 'locality'`) | Unikátní `(country_code, subdivision_code, locality)`; `subdivision_code` je pro locality povinné, takže unikátnost nepropouští duplicitní `NULL` subdivize |
| `weather_regions` | `uq_weather_regions_subdivision` (partial, `WHERE region_type = 'subdivision'`) | Unikátní `(country_code, subdivision_code)` |
| `weather_records` | `idx_weather_records_week_start` | Řazení / filtrování historických týdnů; lookup per segment primárně přes `UNIQUE (weather_region_id, week_start)` |
| `weather_climate_months` | `uq_weather_climate_months_region_month` (`weather_region_id`, `month`) | Unikátní klimatický normál per oblast × měsíc; lookup 12 měsíců oblasti i přes leading FK sloupec |
| `segments` | `idx_segments_trip_start_time` (`trip_id`, `start_time`) | Lineární timeline itineráře; řazení `ORDER BY start_time, id` |
| `segments` | `idx_segments_start_place_id` | Dotčené výlety při změně místa, `country_code` nebo invalidaci cache |
| `segments` | `idx_segments_end_place_id` (partial, `WHERE end_place_id IS NOT NULL`) | Totéž pro cílová místa segmentů |
| `segments` | `idx_segments_segment_kind` | Filtrování dle typu segmentu |
| `segments` | `idx_segments_difficulty` | Dotazy na úrovni segmentu; katalog výletů preferuje `trips.max_difficulty` |
| `segment_images` | `idx_segment_images_segment_sort` (`segment_id`, `sort_order`) | Galerie segmentu v pořadí `ORDER BY sort_order, id` |
| `transit_details` | `idx_transit_details_transport_mode` | Filtrování robotaxi vs. ostatní doprava |
| `transit_details` | `idx_transit_details_provider_id` | Filtrování výletů dle providera (např. všechny výlety s Waymo) |
| `transit_details` | `idx_transit_details_vehicle_model_id` (partial, `WHERE vehicle_model_id IS NOT NULL`) | Dotazy na model vozu |
| `transit_details` | `idx_transit_details_pickup_zone` (partial, `WHERE pickup_zone_place_id IS NOT NULL`) | Pickup zóny |
| `transit_details` | `idx_transit_details_dropoff_zone` (partial, `WHERE dropoff_zone_place_id IS NOT NULL`) | Dropoff zóny |
| `robotaxi_providers` | `idx_robotaxi_providers_active` (partial, `WHERE is_active = true`) | Jen aktivní providery |
| `robotaxi_vehicle_models` | `uq_robotaxi_vehicle_models_provider_slug` (`provider_id`, `slug`) | Unikátní slug per provider |
| `robotaxi_vehicle_models` | `idx_robotaxi_vehicle_models_provider_id` | Modely daného providera |
| `robotaxi_vehicle_models` | `idx_robotaxi_vehicle_models_active` (partial, `WHERE is_active = true`) | Jen aktivní modely |
| `provider_service_areas` | `uq_provider_service_areas_geo` (`provider_id`, `country_code`, `subdivision_code`, `locality`) | Unikátní oblast per provider |
| `provider_service_areas` | `idx_provider_service_areas_provider_id` | Oblasti daného providera |
| `provider_service_areas` | `idx_provider_service_areas_status` | Filtrování dle stavu provozu |
| `provider_service_areas` | `idx_provider_service_areas_locality` (`country_code`, `subdivision_code`, `locality`) | Katalog měst s robotaxi |
| `robotaxi_advisory_items` | `idx_robotaxi_advisory_items_active` (partial, `WHERE is_active = true`) | Jen aktivní položky |
| `robotaxi_advisory_rules` | `idx_robotaxi_advisory_rules_item_id` | Pravidla pro položku |
| `robotaxi_advisory_rules` | `idx_robotaxi_advisory_rules_active` (partial, `WHERE is_active = true`) | Jen aktivní pravidla |
| `robotaxi_advisory_rule_fog_conditions` | PK `(advisory_rule_id, fog_condition)` | Lookup podmínek pravidla |
| `robotaxi_advisory_rule_precipitation_intensities` | PK `(advisory_rule_id, precipitation_intensity)` | Lookup podmínek pravidla |
| `robotaxi_advisory_rule_wind_forces` | PK `(advisory_rule_id, wind_force)` | Lookup podmínek pravidla |
| `segment_robotaxi_advisories` | PK `(segment_id, advisory_item_id)` + `priority` | Cache upozornění per segment |
| `trip_robotaxi_advisories` | PK `(trip_id, advisory_item_id)` | Agregovaná cache upozornění výletu |
| `clothing_items` | `idx_clothing_items_category` | Filtrování / skupiny v UI |
| `clothing_items` | `idx_clothing_items_active` (partial, `WHERE is_active = true`) | Jen aktivní položky |
| `clothing_rules` | `idx_clothing_rules_item_id` | Pravidla pro položku |
| `clothing_rules` | `idx_clothing_rules_active` (partial, `WHERE is_active = true`) | Jen aktivní pravidla |
| `clothing_rule_precipitation_intensities` | PK `(clothing_rule_id, precipitation_intensity)` | Lookup podmínek pravidla |
| `clothing_rule_wind_forces` | PK `(clothing_rule_id, wind_force)` | Lookup podmínek pravidla |
| `clothing_rule_fog_conditions` | PK `(clothing_rule_id, fog_condition)` | Lookup podmínek pravidla |
| `clothing_rule_sky_conditions` | PK `(clothing_rule_id, sky_condition)` | Lookup podmínek pravidla |
| `clothing_rule_difficulties` | PK `(clothing_rule_id, segment_difficulty)` | Lookup podmínek pravidla |
| `trip_packing_items` | PK `(trip_id, clothing_item_id)` | Agregovaná cache balení výletu |
| `trip_packing_item_sources` | PK `(trip_id, clothing_item_id, source)` | Zdroje doporučení |
| `segment_packing_items` | PK `(segment_id, clothing_item_id)` + `priority` | Cache balení per segment |
| `travel_requirement_items` | `idx_travel_requirement_items_active` (partial, `WHERE is_active = true`) | Jen aktivní položky |
| `travel_requirement_rules` | `idx_travel_requirement_rules_item_id` | Pravidla pro položku |
| `travel_requirement_rules` | `idx_travel_requirement_rules_active` (partial, `WHERE is_active = true`) | Jen aktivní pravidla |
| `trip_travel_requirements` | PK `(trip_id, travel_requirement_item_id)` | Agregovaná cache cestovních požadavků výletu |
| `trip_travel_requirement_sources` | PK `(trip_id, travel_requirement_item_id, source)` + `priority` | Zdroje doporučení a jejich priority |

---

