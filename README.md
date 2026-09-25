# CRE Concentration Radar

Screens every U.S. bank against the interagency commercial real estate (CRE) concentration tests and stress-tests each bank's CRE book against the 2026 refinancing wall. It runs on Cloudflare: Workers for the API, D1 for the database, and static assets for the dashboard.

## Why this exists

The Federal Reserve's June 2026 Supervision and Regulation Report says CRE delinquencies remain above their decade averages and that examiners are focusing on banks with heavy CRE concentrations, especially office and multifamily, including their stress-testing assumptions and allowance methods. Community and regional banks carry most of this exposure. This tool shows which banks cross the supervisory screening thresholds and what a rate, cap rate and income shock would do to their capital.

## What it shows

- **Quadrant screen**: every bank plotted by CRE-to-capital against 36-month CRE growth, with the 300% and 50% thresholds marked.
- **Distribution**: how concentration is spread across banks, filterable by state.
- **Watchlist**: sortable table of the most concentrated banks.
- **Bank detail**: CRE composition, 13-quarter concentration trend, and a stress simulator.

## How it's built

```
Browser ──> Cloudflare Worker ──> D1 (SQLite at the edge)
              │
              └── static files in /public (dashboard)
```

| Path | What it does |
|---|---|
| `pipeline/build_dataset.py` | Parses FFIEC Call Report bulk files (or generates demo data) and writes `db/seed.sql` |
| `db/schema.sql` | D1 tables: `banks`, `financials`, `meta` |
| `src/worker.js` | API: `/api/meta`, `/api/screen?quarter=`, `/api/bank/:rssd`, with edge caching |
| `public/index.html` | Dashboard: ECharts visualizations and the stress model |
| `wrangler.jsonc` | Cloudflare configuration |

## Setup

You need Node.js 18 or newer, Python 3.9 or newer, a free Cloudflare account, and a GitHub account.

### 1. Install and log in

```bash
npm install
npx wrangler login          # opens the browser to authorize your Cloudflare account
```

### 2. Create the database

```bash
npx wrangler d1 create cre-radar
```

Copy the `database_id` it prints into `wrangler.jsonc`, replacing `REPLACE_WITH_YOUR_DATABASE_ID`.

### 3. Run locally with demo data

```bash
npm run seed:demo           # writes db/seed.sql with 650 synthetic banks
npm run db:local            # loads schema + seed into a local D1
npm run dev                 # http://localhost:8787
```

### 4. Deploy

```bash
npm run db:remote           # loads the same data into your real D1 database
npm run deploy              # publishes to https://cre-radar.<your-subdomain>.workers.dev
```

### 5. Connect GitHub (optional but recommended)

Push this folder to a GitHub repo. In the Cloudflare dashboard, open **Workers & Pages**, select `cre-radar`, go to **Settings > Build**, and connect the repo. Every push to `main` then redeploys the site. Data loads (`npm run db:remote`) stay a manual step you run each quarter.

## Loading real FDIC data (easiest)

```bash
python pipeline/fetch_fdic.py        # Mac: python3
npx wrangler d1 execute cre-radar --remote --file=db/schema.sql
npx wrangler d1 execute cre-radar --remote --file=db/seed.sql
npx wrangler deploy
```

`fetch_fdic.py` calls the free FDIC BankFind Suite API (no key needed), finds the latest published quarter, checks which fields are available, downloads 13 quarters for every insured bank, and writes `db/seed.sql`. Banks are identified by FDIC certificate number.

## Loading real FFIEC data (alternative)

1. Open the FFIEC Central Data Repository bulk download page: https://cdr.ffiec.gov/public/PWS/DownloadBulkData.aspx
2. Choose **Call Reports -- Single Period**, tab-delimited format.
3. Download **13 consecutive quarters**. The 36-month growth test compares the latest quarter with the one 12 quarters earlier.
4. Unzip each quarter into `data/raw/YYYYQn/`, for example `data/raw/2026Q2/`.
5. Run:

```bash
npm run seed:ffiec
npm run db:local            # check it locally first
npm run db:remote
```

The API caches responses at the edge for an hour, so a fresh load can take up to an hour to show on the live site.

**Verify the item codes.** The pipeline maps these MDRM items. Check them against the FFIEC MDRM data dictionary for the quarters you load, because codes occasionally change:

| Field | MDRM item | Schedule |
|---|---|---|
| Total assets | 2170 | RC |
| Total loans | 2122 | RC-C Part I |
| 1–4 family construction | F158 | RC-C Part I |
| Other construction and land | F159 | RC-C Part I |
| Multifamily | 1460 | RC-C Part I |
| Owner-occupied nonfarm nonresidential | F160 | RC-C Part I |
| Other nonfarm nonresidential | F161 | RC-C Part I |
| CRE not secured by real estate | 2746 | RC-C Part I, Memoranda |
| Tier 1 capital | 8274 | RC-R Part I |
| Allowance for credit losses on loans | 3123 | RC |

Consolidated (RCFD) values are used when a bank files them; otherwise domestic (RCON).

## Method

**Concentration.** Follows the 2006 interagency guidance on concentrations in CRE lending. Test CRE is construction and land development, multifamily, non-owner-occupied nonfarm nonresidential, and CRE loans not secured by real estate. Capital is Tier 1 capital plus the allowance for credit losses. Flags:

- CRE at 300% or more of capital **and** 50% or more growth over 36 months
- Construction and land development at 100% or more of capital

These are screening criteria that trigger closer supervisory review, not lending limits.

**Stress model (illustrative).** For each CRE segment:

- Stressed DSCR = baseline DSCR × (1 − income decline) ÷ (1 + share refinancing × (new payment ÷ old payment − 1)), where loans reprice from 4.5% to 6.75% plus the rate shock on a 25-year amortization.
- Stressed LTV = baseline LTV × (new cap rate ÷ old cap rate) ÷ (1 − income decline).
- PD = share of loans with DSCR below 1.0, assuming loan-level DSCR is lognormal around the stressed median.
- LGD = 1 − 0.85 ÷ (stressed LTV × tail factor), floored at 10%, where the tail factor reflects defaulted loans carrying above-average leverage.
- Expected loss = exposure × PD × LGD, compared against the allowance and against Tier 1 capital plus allowance.

Segment assumptions live in the `SEGMENTS` array in `public/index.html`. They are placeholders chosen to produce loss rates in a plausible range, not calibrated estimates. A good next step is calibrating them to published supervisory stress-test loss rates.

## Roadmap

- **Automated quarterly refresh**: a Cron Trigger that checks for a new FFIEC release and alerts you through a Queue.
- **Peer comparison**: rank a bank against its asset-size peers in its state.
- **Maturity overlay**: add public CMBS maturity data by property type to weight the "loans refinancing" assumption.
- **Written findings**: a one-page memo per state, for example "Oklahoma banks above the 300% threshold and their stressed capital", generated from the data.

## Disclaimer

Demo mode uses synthetic banks. With real data, results are an analytical screen built from public filings, not a supervisory assessment or investment advice.
