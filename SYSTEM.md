# Bilhandel Banner Admin — Systemdokumentation

## Overblik

Et selvstændigt bannersystem til Bilhandel.dk, der håndterer salg, levering og statistik for displayannoncer. Systemet er bygget på Supabase (database + edge functions) og en enkelt HTML-fil som admin-interface.

**Supabase projekt:** `mbqeozymspvyouozpipv` (eu-west-1)
**GitHub:** https://github.com/soerensteen/Bilhandel-banner-booking
**Admin UI:** `index.html`

---

## Arkitektur

```
Annoncør-website
    │
    ├─ <script src=".../serve-ad-script">   ← indlæser JS-snippet
    │         │
    │         ├─ fetch(".../serve-ad?placement=&device=")   ← henter annonce
    │         │         │
    │         │         ├─ læser: serving_config (1 query, pre-beregnet)
    │         │         └─ skriver: raw_impressions (1 insert)
    │         │
    │         └─ onClick → GET .../track-click?campaign_id=&placement=&device=
    │                   └─ skriver: raw_clicks (1 insert)
    │
Admin UI (index.html)
    └─ Supabase JS SDK → direkte til database

pg_cron (hvert 15. min)
    └─ HTTP POST → aggregate-stats
              ├─ aggregerer raw_impressions → daily_stats
              ├─ aggregerer raw_clicks → daily_clicks
              ├─ opdaterer campaign_placements.used
              ├─ pre-beregner serving_config (pacing-vægte)
              └─ opdaterer campaign statuser automatisk
```

---

## Database

### `campaigns`
Hoveddatatabellen for en kampagne.

| Kolonne | Type | Beskrivelse |
|---|---|---|
| `id` | uuid | Primærnøgle |
| `customer` | text | Annoncørens navn |
| `status` | text | `active`, `scheduled`, `paused`, `ended` |
| `start_date` | date | Kampagnestart |
| `end_date` | date | Kampagneslut |
| `notes` | text | Interne noter |

### `campaign_placements`
Definerer hvilket placement en kampagne køber, og hvad budgettet er.

| Kolonne | Type | Beskrivelse |
|---|---|---|
| `campaign_id` | uuid | FK → campaigns |
| `placement` | text | `forside`, `srp`, `vip` |
| `budget` | integer | Antal impressions købt (per måned) |
| `used` | integer | Leverede impressions denne periode (opdateres af aggregate-stats) |

### `campaign_creatives`
Selve annoncematerialet — én række per placement/device-kombination.

| Kolonne | Type | Beskrivelse |
|---|---|---|
| `campaign_id` | uuid | FK → campaigns |
| `placement` | text | `forside`, `srp`, `vip` |
| `device` | text | `desktop`, `mobile`, `all` |
| `image_url` | text | Bannerbillede |
| `headline` | text | Overskrift |
| `body_text` | text | Brødtekst |
| `click_url` | text | Klik-destination |

### `serving_config`
Pre-beregnede pacing-vægte og creative data. Genberegnes hvert 15. minut af aggregate-stats. Bruges af serve-ad til hurtig opslag (1 query i stedet for 5).

| Kolonne | Type | Beskrivelse |
|---|---|---|
| `id` | uuid | Primærnøgle |
| `placement` | text | `forside`, `srp`, `vip` |
| `campaign_id` | uuid | FK → campaigns |
| `weight` | float | Pre-beregnet pacing-vægt (0 = foran plan) |
| `creative` | jsonb | `{customer, device, image_url, headline, body_text, click_url}` |
| `updated_at` | timestamptz | Hvornår rækken sidst blev beregnet |

### `raw_impressions`
Hvert enkelt visning logges her i realtid.

| Kolonne | Type | Beskrivelse |
|---|---|---|
| `id` | bigint | Auto-increment (bruges som cursor) |
| `campaign_id` | uuid | FK → campaigns |
| `placement` | text | |
| `device` | text | |
| `recorded_at` | timestamptz | Tidsstempel (default: now()) |

### `raw_clicks`
Hvert enkelt klik logges her i realtid.

| Kolonne | Type | Beskrivelse |
|---|---|---|
| `id` | bigint | Auto-increment (bruges som cursor) |
| `campaign_id` | uuid | FK → campaigns |
| `placement` | text | |
| `device` | text | |
| `recorded_at` | timestamptz | Tidsstempel (default: now()) |

### `daily_stats`
Aggregerede impressions per kampagne/placement/dag.

| Kolonne | Type | Beskrivelse |
|---|---|---|
| `campaign_id` | uuid | FK → campaigns |
| `placement` | text | |
| `stat_date` | date | |
| `impression_count` | integer | Antal impressions denne dag |

### `daily_clicks`
Aggregerede klik per kampagne/placement/dag.

| Kolonne | Type | Beskrivelse |
|---|---|---|
| `campaign_id` | uuid | FK → campaigns |
| `placement` | text | |
| `stat_date` | date | |
| `click_count` | integer | Antal klik denne dag |

### `aggregation_cursor`
Holder styr på, hvilke `raw_impressions` og `raw_clicks` der er behandlet. Én række (id=1).

| Kolonne | Type | Beskrivelse |
|---|---|---|
| `last_processed_id` | bigint | Højeste raw_impressions.id der er aggregeret |
| `last_processed_click_id` | bigint | Højeste raw_clicks.id der er aggregeret |
| `last_run_at` | timestamptz | Hvornår aggregate-stats sidst kørte |

---

## Edge Functions

### `serve-ad`
**URL:** `GET /functions/v1/serve-ad?placement=<p>&device=<d>`
**Auth:** Ingen (verify_jwt: false)
**Queries:** 1 read + 1 write (optimeret fra 5 queries)

Leverer en annonce baseret på pre-beregnede pacing-vægte fra `serving_config`.

**Algoritme:**
1. `SELECT` fra `serving_config` WHERE placement = X (1 query, indekseret)
2. Filtrer på device i hukommelsen
3. **SRP:** Returner *alle* valgbare kampagner (multi-format)
4. **Forside/VIP:** Vælg én kampagne via vægtet tilfældig (større weight = højere chance)
5. `INSERT` impression i `raw_impressions`

**Svar (single):**
```json
{
  "type": "single",
  "ad": {
    "campaign_id": "...",
    "customer": "Toyota",
    "image_url": "https://...",
    "headline": "Ny model klar",
    "body_text": "Se tilbud her",
    "click_url": "https://toyota.dk"
  }
}
```

**Svar (multi, SRP):**
```json
{
  "type": "multi",
  "ads": [{ "campaign_id": "...", "logo_url": "...", ... }]
}
```

---

### `track-click`
**URL:** `GET /functions/v1/track-click?campaign_id=<id>&placement=<p>&device=<d>`
**Auth:** Ingen (verify_jwt: false)
**Svar:** 204 No Content

Logger et klik i `raw_clicks`. Kaldes fra serve-ad-script via et usynligt Image-request når brugeren klikker på et banner.

---

### `serve-ad-script`
**URL:** `GET /functions/v1/serve-ad-script`
**Auth:** Ingen (verify_jwt: false)
**Cache:** 5 minutter (`Cache-Control: public, max-age=300`)
**Kildekode:** Deployed som Supabase edge function (ikke en fil i repoet)

Returnerer et dynamisk genereret JavaScript-snippet. JS'en genereres on-the-fly i edge functionen fordi den injecter Supabase URL'en (`SUPABASE_URL` env var) som API-base i det udleverede script.

**Hvad scriptet gør på annoncør-websitet:**
1. Finder alle `<ins class="bh-banner">` elementer på siden
2. Laver et `fetch()` kald til `serve-ad` per element (med placement + device fra data-attributter)
3. Renderer banner HTML (billede-link, tekst-link, eller fallback med kundenavn)
4. Tilføjer click event listeners der kalder `track-click` via usynligt Image-request

**Sådan integreres det på et website:**
```html
<!-- Placer dette element hvor banneret skal vises -->
<ins class="bh-banner" data-placement="forside" data-device="desktop" style="display:block;width:728px;height:90px"></ins>

<!-- Indsæt scriptet i bunden af <body> -->
<script src="https://mbqeozymspvyouozpipv.supabase.co/functions/v1/serve-ad-script" async></script>
```

**Tilgængelige placements:** `forside`, `srp`, `vip`
**Tilgængelige devices:** `desktop`, `mobile` (SRP bruger altid `all`)

---

### `aggregate-stats`
**URL:** `POST /functions/v1/aggregate-stats`
**Auth:** Ingen (verify_jwt: false)
**Kaldes af:** pg_cron hvert 15. minut

Behandler op til 10.000 nye `raw_impressions` og `raw_clicks`, og pre-beregner serving config.

**Trin:**
1. Læs `aggregation_cursor` (last_processed_id + last_processed_click_id)
2. Hent og aggreger `raw_impressions` → `daily_stats`
3. Hent og aggreger `raw_clicks` → `daily_clicks`
4. Ryk cursors frem
5. Genberegn `campaign_placements.used` for indeværende måned
6. Auto-opdater kampagnestatuser:
   - `scheduled` → `active` hvis `start_date <= i dag <= end_date`
   - `active` → `ended` hvis `end_date < i dag`
7. Pre-beregn `serving_config`:
   - Beregn pacing-vægte for alle aktive kampagner per placement
   - Gem creative data som jsonb
   - Atomisk replace (delete + insert)

---

## Automatisk kørsel (pg_cron)

`aggregate-stats` er sat op til at køre automatisk hvert 15. minut via pg_cron + pg_net:

```sql
-- Job navn: aggregate-stats-every-15min
-- Cron: */15 * * * *
-- Kalder edge function via pg_net HTTP POST
```

**Tjek job-status:**
```sql
SELECT * FROM cron.job;
SELECT * FROM cron.job_run_details ORDER BY start_time DESC LIMIT 10;
```

---

## Admin UI (`index.html`)

Mørk-temaet single-page app der kommunikerer direkte med Supabase via JS SDK.

**Funktioner:**
- Oversigt over alle kampagner med status og pacing
- Opret/rediger kampagner
- Tilknyt placements med budget
- Upload og rediger creatives (billede, headline, tekst, klik-URL)
- Realtids statistik pr. kampagne og placement

---

## Dataflow: Fra impression til statistik

```
1. Besøgende ser annonce på bilhandel.dk
        ↓
2. serve-ad læser serving_config → returnerer annonce
   serve-ad logger → raw_impressions
        ↓
3. Besøgende klikker på annonce
   serve-ad-script kalder → track-click → raw_clicks
        ↓
4. Hvert 15. min kører aggregate-stats:
   - raw_impressions → daily_stats
   - raw_clicks → daily_clicks
   - Genberegner campaign_placements.used
   - Pre-beregner serving_config (pacing-vægte)
        ↓
5. Næste gang serve-ad kører:
   - Læser serving_config (1 query)
   - Pacing allerede beregnet — vælger kampagne direkte
```

---

## Pacing-formel

Beregnes i `aggregate-stats` og gemmes i `serving_config.weight`:

```
periodStart   = max(campaign.start_date, første dag i måneden)
periodEnd     = min(campaign.end_date, sidste dag i måneden)
totalDays     = dage i perioden
daysElapsed   = dage fra periodStart til i dag

expectedByNow = budget × (daysElapsed / totalDays)
deficit       = expectedByNow − delivered

weight        = max(0, deficit)
  → 0 = kampagnen er foran plan, serveres ikke
  → positiv = kampagnen er bagud, vægtet sandsynlighed for at blive valgt
```

---

## Performance

Systemet er optimeret til 5M+ sidevisninger/måned:

| Metric | Før | Efter |
|---|---|---|
| DB queries per serve-ad | 5 (campaigns join + daily_stats + cursor + raw_impressions scan + insert) | 2 (serving_config read + raw_impressions insert) |
| Pacing-beregning | Realtime per request | Pre-beregnet hvert 15. min |
| Pacing-præcision | Real-time | 15 min forsinkelse (acceptabelt for bannerannoncer) |
