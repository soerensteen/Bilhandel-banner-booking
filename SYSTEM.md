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
    │         └─ fetch(".../serve-ad?placement=&device=")   ← henter annonce
    │                   │
    │                   ├─ læser: campaigns, campaign_placements, campaign_creatives
    │                   ├─ læser: daily_stats + raw_impressions (pacing)
    │                   └─ skriver: raw_impressions (impression tracking)
    │
Admin UI (index.html)
    └─ Supabase JS SDK → direkte til database

pg_cron (hvert 15. min)
    └─ HTTP POST → aggregate-stats
              ├─ aggregerer raw_impressions → daily_stats
              ├─ opdaterer campaign_placements.used
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

### `raw_impressions`
Hvert enkelt visning logges her i realtid.

| Kolonne | Type | Beskrivelse |
|---|---|---|
| `id` | bigint | Auto-increment (bruges som cursor) |
| `campaign_id` | uuid | FK → campaigns |
| `placement` | text | |
| `device` | text | |
| `recorded_at` | timestamptz | Tidsstempel (default: now()) |

### `daily_stats`
Aggregerede impressions per kampagne/placement/dag. Bruges til pacing.

| Kolonne | Type | Beskrivelse |
|---|---|---|
| `campaign_id` | uuid | FK → campaigns |
| `placement` | text | |
| `stat_date` | date | |
| `impression_count` | integer | Antal impressions denne dag |

### `aggregation_cursor`
Holder styr på, hvilke `raw_impressions` der er behandlet. Én række (id=1).

| Kolonne | Type | Beskrivelse |
|---|---|---|
| `last_processed_id` | bigint | Højeste raw_impressions.id der er aggregeret |
| `last_run_at` | timestamptz | Hvornår aggregate-stats sidst kørte |

---

## Edge Functions

### `serve-ad`
**URL:** `GET /functions/v1/serve-ad?placement=<p>&device=<d>`
**Auth:** Ingen (verify_jwt: false)

Leverer en annonce til et givent placement. Implementerer et **pacing-algoritme** der sikrer jævn fordeling af impressions over måneden.

**Algoritme:**
1. Find aktive kampagner med budget for det ønskede placement
2. Hent aggregerede impressions (`daily_stats`) + ikke-aggregerede (`raw_impressions` siden cursor)
3. Beregn forventet levering: `budget × (daysElapsed / totalDays)`
4. `deficit = forventet − leveret` — positivt = bagud i pacing
5. Kampagner med `deficit > 0` er valgbare
6. **SRP:** Returner *alle* valgbare kampagner (multi-format)
7. **Forside/VIP:** Vælg én kampagne via vægtet tilfældig (større deficit = højere chance)
8. Log impression i `raw_impressions`

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

### `serve-ad-script`
**URL:** `GET /functions/v1/serve-ad-script`
**Auth:** Ingen (verify_jwt: false)

Returnerer et JavaScript-snippet, som annoncør-websites inkluderer. Scriptet finder alle `<ins class="bh-banner">` elementer og fylder dem med annoncer via `serve-ad`.

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

Behandler op til 10.000 nye `raw_impressions` og aggregerer dem ind i `daily_stats`.

**Trin:**
1. Læs `aggregation_cursor.last_processed_id`
2. Hent alle `raw_impressions` med `id > last_processed_id`
3. Grupper efter `campaign_id + placement + dato`
4. Upsert ind i `daily_stats` (opdater eksisterende rækker eller opret nye)
5. Rykke cursor frem til højeste behandlede id
6. Genberegn `campaign_placements.used` for alle aktive/schedulede kampagner (indeværende måned)
7. Auto-opdater kampagnestatuser:
   - `scheduled` → `active` hvis `start_date <= i dag <= end_date`
   - `active` → `ended` hvis `end_date < i dag`

---

## Automatisk kørsel (pg_cron)

`aggregate-stats` er sat op til at køre automatisk hvert 15. minut via pg_cron:

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
2. serve-ad logger → raw_impressions (id: 42, campaign_id: X, placement: forside)
        ↓
3. Hvert 15. min kører aggregate-stats:
   - Behandler raw_impressions id > last_cursor
   - Lægger til daily_stats (campaign X, forside, dato: 2026-03-12, count++)
   - Opdaterer aggregation_cursor.last_processed_id = 42
   - Opdaterer campaign_placements.used
        ↓
4. Næste gang serve-ad kører:
   - Læser daily_stats for leverede impressions
   - Læser raw_impressions > cursor for ikke-aggregerede
   - Beregner pacing og beslutter om kampagnen skal serveres
```

---

## Pacing-formel

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
