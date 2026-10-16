# Free Google Maps Scraper — Extract Business Data Without a Browser

[![Python](https://img.shields.io/badge/python-3.11%2B-blue)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![No browser](https://img.shields.io/badge/browser-not%20required-success)]()
[![API key](https://img.shields.io/badge/API%20key-not%20required-success)]()
[![Cost](https://img.shields.io/badge/cost-free-success)]()

**Google Maps Scraper** is a free, open-source Python tool that extracts
business data from Google Maps using **pure HTTP requests — no browser, no
Selenium, no Playwright, no API key**. Give it a category and a location
(or any search query), and it produces a clean **CSV** of the matching
businesses: name, category, address, phone number, website, **email
address**, rating, review count, opening hours, GPS coordinates and more.

It talks directly to the same internal endpoint the Google Maps frontend
uses, so it is dramatically faster than browser-based scrapers, never
downloads a Chrome driver, and works on any machine — a laptop, a VPS or a
Raspberry Pi.

> ⚠️ **Educational purpose only** — see the
> [Disclaimer](#-disclaimer-educational-purpose-only) before using this
> project.

## 📊 Sample output

Real rows scraped with this tool (`dentist in Austin, TX`):

| name | phone | rating | reviews | website | email |
|---|---|---|---|---|---|
| ATX Family Dental | (512) 717-3147 | 4.9 | 913 | atxfamilydental.com | info@atxfamilydental.com |
| Austin Dental Works | (512) 877-9822 | 4.9 | 632 | austindentalworks.com | info@austindentalworks.com |
| Austin Lifetime Dental | (512) 835-1924 | 4.9 | — | lifetimedental.com | info@lifetimedental.com |

Every business is exported as one CSV row with **19 data fields** — ready
for Excel, Google Sheets, a CRM, or Python/pandas. Built for local **lead
generation**, market research and competitive analysis.

## ✨ Why this Google Maps scraper?

Most open-source Google Maps scrapers drive a headless browser: they are
slow, break when Google changes its DOM, and crash with driver/install
errors. This one takes a different approach:

- **No browser at all** — it calls Google Maps' internal `tbm=map` endpoint
  directly over HTTP with Chrome-grade TLS impersonation. A page of 20
  results costs ~1 second instead of ~30.
- **Nothing to install but Python** — no chromedriver, no Playwright
  binaries, no Docker. `pip install -r requirements.txt` and you're done.
- **Beats the ~120-results-per-query cap** — [grid mode](#-scrape-all-businesses-in-a-city-grid-mode)
  tiles a city into small map cells and scrapes every cell, so you cover
  every corner of a city, not just the first page(s).
- **Rich data, not just links** — place details (hours, review counts,
  plus codes, descriptions) and **contact emails extracted from each
  business's website** come standard.
- **Built to run unattended** — rotating proxies with escalating
  cooldowns, per-proxy rate limiting, a global circuit breaker, exponential
  backoff, and crash-safe resume: interrupt it with Ctrl+C, re-run the same
  command, and it continues exactly where it stopped.

## 🚀 Quick start — how to scrape Google Maps in 3 commands

**1. Install** (Python 3.11 or newer):

```bash
git clone https://github.com/sonagara-vashram/google-maps-scraper.git
cd google-maps-scraper
pip install -r requirements.txt
```

**2. Scrape** — one category in one location:

```bash
python main.py scrape --category "dentist" --location "Austin, TX"
```

**3. Open `businesses.csv`** — the matching businesses with phone, website,
email, rating, hours and coordinates, one row each.

That's the whole workflow. No API key, no credits, no sign-up.

Other one-liners:

```bash
# any free-form search query
python main.py scrape --query "vegan restaurants in Berlin"

# ALL dentists in Austin, not just the first ~120 (grid mode)
python main.py scrape --category "dentist" --location "Austin, TX" --all

# through residential proxies (recommended at scale)
python main.py scrape --category "dentist" --location "Austin, TX" \
    --proxy-file config/proxies.txt
```

## 📦 What data can you extract?

Each business is one CSV row with these fields:

| Field | Description |
|---|---|
| `name` | Business name as shown on Google Maps |
| `category` | Business categories (primary first) |
| `address` | Formatted street address |
| `phone` | Display phone number |
| `phone_intl` | E.164 phone number (`+15127173147`) — ready for dialers |
| `website` | Business website URL |
| `email` | Contact email extracted from the business website |
| `rating` | Average star rating (1.0–5.0) |
| `review_count` | Total number of Google reviews |
| `lat`, `lng` | GPS coordinates |
| `plus_code` | Google Plus Code (open location code) |
| `hours` | Opening hours per weekday |
| `price_level` | Price range (`$`–`$$$$`) where the category has one |
| `business_status` | `OPERATIONAL` / `CLOSED_TEMPORARILY` / `CLOSED_PERMANENTLY` |
| `google_maps_url` | Stable `maps.google.com/?cid=...` deep link |
| `place_id` | Google place id (`ChIJ...`) — joins with the Places API |
| `query` | The search that surfaced this business (provenance) |
| `added` | UTC timestamp when first seen |

Field availability varies by business: Google omits what a listing
doesn't have (rating, hours, price level, ...), and its details endpoint
occasionally serves a reduced payload — the scraper retries once and keeps
whatever Google returns.

With `--reviews` a second CSV (`reviews.csv`) adds reviewer name, rating,
text, date and owner replies per business. Review-text availability depends
on your region/session — Google gates review content for signed-out
requests in some regions, in which case that file simply stays empty while
`rating` and `review_count` still work.

## 🗺️ Scrape ALL businesses in a city (grid mode)

Google Maps stops at roughly **120 results per search** — a hard UI cap
that hides most businesses in any real city. This scraper's `--all` mode
solves it:

```bash
python main.py scrape --category "dentist" --location "Austin, TX" --all
```

How it works:

1. The location is **geocoded** once (OpenStreetMap Nominatim — free, no
   key) to get the city's bounding box.
2. The box is **tiled with overlapping cells** (default 2 km, 25% overlap).
3. Each cell runs its own viewport-scoped search with full pagination.
4. Results are **deduplicated by place id**, so businesses near cell
   borders appear exactly once in your CSV.

Tune the coverage with `--cell-km 1.0` (more thorough, slower) or bound a
metro area with `--radius-km 10`.

## 📧 Extract emails from Google Maps listings

Google Maps doesn't publish email addresses — so this scraper fetches each
business's **website** and extracts its contact email (mailto links plus
plain-text addresses, with junk filtering). It's on by default; the best
candidate email lands in the `email` column of your CSV.

```bash
python main.py scrape --category "dentist" --location "Austin, TX" --no-emails   # skip it
```

## 🔁 Scrape Google Maps reviews (best-effort)

```bash
python main.py scrape --category "dentist" --location "Austin, TX" --reviews
```

Writes `reviews.csv` (reviewer, rating, text, date, owner reply). Honest
note: Google gates full review text for signed-out sessions in some
regions — there the reviews file stays empty while ratings and review
counts are unaffected.

## 🛡️ How to scrape Google Maps without getting blocked

Google rate-limits per IP. The scraper ships with the full defensive stack
so you don't have to think about it — add proxies and it handles the rest:

- **Proxy rotation** — one healthy proxy is leased per query, human-like
  randomised delays between pages.
- **Escalating cooldowns** — a blocked proxy sits out 10 → 20 → … minutes;
  the query continues on a fresh identity.
- **Circuit breaker** — if the block rate spikes, *everything* pauses
  briefly instead of burning your whole pool.
- **No real-IP fallback** — if every proxy is cooling down, the run halts
  cleanly (state saved, resumable) instead of firing requests from your
  home IP.

Give it a plain text file, one proxy per line:

```text
http://user:password@1.2.3.4:8080
socks5://user:password@gateway.provider.com:1080
```

Residential/ISP proxies work best; datacenter IPs get flagged much faster.
Before a real run, verify your list: `python check_proxies.py` — it checks
every proxy in `config/proxies.txt` and reports exit IP + latency (add
`--maps` to also probe Google Maps reachability per proxy).
Scale honestly: match `--workers` roughly to your proxy count (1–2 workers
per proxy) and keep `--min-interval` at 1.5s or above.

## ⚙️ Commands & options

| Command | What it does |
| --- | --- |
| `python main.py scrape ...` | One category+location (or query) → CSV. **Start here.** |
| `python main.py run ...` | Advanced: crawl a (category × city) grid from Census state files |
| `python main.py report` | Stored progress stats — no network |
| `python main.py export --out delivery.csv` | Regenerate a clean, dedupe-free CSV from the state DB |
| `python main.py validate ...` | Preflight: config problems + pending job count |

The options that matter most (see `--help` for all):

| Flag | Default | Meaning |
| --- | --- | --- |
| `--category` / `--location` | — | Business type + place, e.g. `"dentist"` / `"Austin, TX"` |
| `--query` | — | Free-form search instead of category/location |
| `--all` | off | Grid mode: tile the location into cells for full coverage |
| `--cell-km` / `--radius-km` | `2` / `0` | Grid cell size / radius cap |
| `--proxy-file` | `config/proxies.txt` | Proxy list; missing file = direct connection |
| `--workers` | `2` | Parallel workers |
| `--max-pages` | `10` | Pages (20 results each) per query; `0` = unlimited |
| `--no-details` / `--no-emails` / `--reviews` | details+emails on | Enrichment depth |
| `--lang` / `--gl` | `en` / `us` | Results language / country |
| `--out` / `--reviews-out` | `businesses.csv` / `reviews.csv` | Output paths |
| `--db` | `state.sqlite3` | Resume-state file path |
| `--fresh` | off | Wipe state and start over |
| `--research` | off | Re-run finished queries and backfill missing details/emails |

## 💾 Resume, dedupe and durability

- **Resume** — every query is checkpointed in `state.sqlite3` (SQLite
  WAL). Ctrl+C, crash, reboot: re-run the same command and the searches
  continue where they stopped; add `--research` to also re-enrich places
  whose details/emails were mid-flight.
- **Dedupe** — businesses are keyed by their Google place cid, so one
  business found under ten categories is stored — and exported — once.
- **Complete CSVs** — the live CSV gains a row only after a place's
  enrichment finishes, so at the end of every run the deliverables are
  rewritten complete from the database; `export` does the same on demand.

## 🧠 How it works (no browser, pure HTTP)

The scraper requests
`https://www.google.com/search?tbm=map&hl=…&gl=…&q={query}&pb={pb}` — the
same internal endpoint the Maps frontend uses — where the `pb` parameter
encodes the search text, a page size of 20 and a pagination offset.
Responses are `)]}'`-prefixed JSON payloads that are parsed into business
records; the place URL is derived from each place's feature id as
`https://maps.google.com/?cid={decimal}`. Place details come from
`/maps/preview/place`, review data from the internal `batchexecute`
endpoint, and contact emails from a plain fetch of each business website.

```
google-maps-scraper/
├── main.py                    # entry point: python main.py <command>
├── check_proxies.py          # simple proxy checker (optional)
├── config/
│   ├── proxies.txt.example    # copy to proxies.txt and add your proxies
│   └── categories.txt         # categories for the advanced city-grid mode
├── google_maps_scraper/
│   ├── cli.py            scrape | run | report | export | validate
│   ├── config.py         every knob, fail-fast validation
│   ├── engine.py         lifecycle: workers -> monitor -> summary
│   ├── worker.py         crawl policy: proxy pick, pacing, pagination,
│   │                     details/reviews/emails enrichment
│   ├── http_client.py    curl_cffi + Chrome TLS impersonation
│   ├── parser.py         business/review/email extraction
│   ├── constants.py      the tbm=map protocol (pb templates, endpoints)
│   ├── grid.py           --all mode: geocode + cell tiling
│   ├── proxy_pool.py     rotation, cooldowns, credential masking
│   ├── rate_limiter.py   per-proxy minimum-interval pacing
│   ├── circuit_breaker.py global pause when blocks spike
│   ├── database.py       SQLite state: searches, places, reviews, blocks
│   ├── output.py         CSV/txt deliverable writers
│   ├── queries.py        locations x categories query plan (advanced mode)
│   ├── models.py         Business / Review / Job dataclasses
│   ├── metrics.py        thread-safe counters
│   └── logsetup.py       console logging with credential masking
```

## 🆚 Google Maps scraper comparison

How this scraper stacks up against the alternatives:

| | **This scraper** (pure HTTP) | Browser-based scrapers (Selenium/Playwright) | Paid SaaS / APIs (Apify, Outscraper, …) |
| --- | --- | --- | --- |
| Cost | **Free, unlimited** | Free | $ per 1k rows / monthly |
| Setup | `pip install`, no drivers | Chrome + drivers + Docker, fragile | Account + API keys |
| Speed per 20 results | **~1s** | ~30s | Fast (server-side) |
| Gets past the ~120-result cap | **Yes (grid mode)** | Rarely | Yes |
| Emails from business websites | **Yes** | Rarely | Paid add-on |
| Runs on a $5 VPS / Raspberry Pi | **Yes** | Barely | n/a (cloud) |
| Data freshness | Live, you control it | Live | Cached / delayed |
| ToS exposure | Same class (unofficial endpoint) | Unofficial | Official-ish |

## ❓ FAQ

**Is this Google Maps scraper really free?**
Yes — MIT-licensed, no API key, no credits, no usage caps. Your only
optional cost is a residential proxy plan if you scrape at scale.

**Do I need an API key?**
No. The scraper uses Google Maps' internal web endpoints, not the Places
API. For a fully ToS-compliant production pipeline, use the official
[Places API](https://developers.google.com/maps/documentation/places/web-service)
instead.

**How many results can I scrape?**
A single search paginates far beyond the UI's ~120-result cap. With
`--all` (grid mode) you cover the whole location cell by cell — the
smaller the cell, the more complete the coverage. In a live test, 25
cells x 20 results collapsed into 115 unique businesses after dedupe.

**Will my IP get blocked?**
Direct connection is fine for a few hundred requests. At scale, use
residential proxies — the scraper rotates them, cools down blocked ones,
pauses itself when the block rate spikes, and never falls back to your
real IP.

**Is scraping Google Maps legal?**
Scraping public data is a legally nuanced area that differs by
jurisdiction, and using Google's undocumented endpoints violates Google's
Terms of Service. This project is published for educational and research
purposes — you are responsible for how you use it and for complying with
the laws and terms that apply to you.

**Does it work for countries other than the US?**
Yes — pass `--lang` and `--gl` (e.g. `--lang de --gl de` for Germany).
Grid mode works worldwide; the advanced state-file city grid currently
ships with US Census data.

## 🗺️ Roadmap

- [ ] Google Maps **photos** dataset (thumbnail URLs are already in the payload)
- [ ] `xlsx` output alongside CSV
- [ ] Review-text regions autodetect

## ⚠️ Disclaimer (Educational Purpose Only)

- This project accesses Google through **undocumented internal endpoints**,
  which **violates Google's Terms of Service**.
- It is published **strictly for educational and research purposes** — to
  study resilient HTTP-client engineering (proxy rotation, rate limiting,
  circuit breakers, crash-safe state machines), **not** to encourage
  scraping at scale.
- **No platform officially allows scraping.** Automated extraction of
  Google Maps data is only permitted through the official Places API.
- You are **solely responsible** for how you use this code, for the proxy
  infrastructure you run it on, and for complying with all applicable laws
  and terms of service in your jurisdiction.
- The authors are **not affiliated with Google** and accept **no liability**
  for any misuse or any consequences arising from it.
- For any production or commercial need, use the official
  [Google Maps Places API](https://developers.google.com/maps/documentation/places/web-service).

## 📄 License

[MIT](LICENSE) — free to use, modify and ship.
