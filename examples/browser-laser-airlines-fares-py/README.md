# Laser Airlines fare discovery (Python)

Discover the available fare families for one public Laser Airlines route and date with a managed Solari cloud browser.

## The problem

Travelers in Venezuela often need to compare domestic airline fares across fragmented, JavaScript-heavy airline booking sites. Conventional server-hosted scrapers can be unreliable, resource-heavy, and difficult to debug when airline UI flows change.

## Why Solari

This example uses a managed Solari cloud browser to run the real Laser Airlines booking flow remotely. It isolates browser workload from the application server, supports stealth and CAPTCHA handling, and can record sessions for debugging when the airline flow changes. It launches exactly one bounded session and does not enable a residential proxy.

## Production insight

The broader production application successfully used this approach to retrieve Laser Airlines fare families for CCS → PMV, including Economy Light, Economy Basic, Economy Plus, and Business Class. The key value is not generic scraping; it is reliable price discovery for a real underserved travel market.

Browser interaction stays in [`main.py`](main.py). The rendered HTML is frozen with `page.content()` and passed to the separate static parser in [`parse.py`](parse.py), making results-page selector changes easier to isolate.

## Responsible use

- Query only public booking flows.
- Keep requests low-frequency and serial.
- Respect airline terms, availability, and rate limits.
- Do not use the example to collect credentials, evade access controls, or perform bulk extraction.

## Setup and run

```bash
cd examples/browser-laser-airlines-fares-py
pip install -r requirements.txt
export SOLARI_API_KEY=slr_live_...  # https://console.getsolari.com
python main.py --origin CCS --destination PMV --departure-date 2026-09-21 --recording
```

The CCS → PMV command above was validated against the public booking flow on August 31, 2026. Omit `--recording` unless you need a replay for debugging.

The program exits with a concise error when the API key is missing, the date is past or unavailable, Laser reports no flights, Solari or the upstream site blocks the session, or the expected results-page selectors no longer match. It never prints the API key.

## Sample output

```json
[
  {
    "airline": "Laser Airlines",
    "flight_number": "QL-904",
    "departure_airport": "CCS",
    "arrival_airport": "PMV",
    "departure_datetime": "2026-09-21T15:30:00",
    "arrival_datetime": "2026-09-21T16:20:00",
    "fare_class": "ECONOMY-LIGHT",
    "price_usd": 50.0,
    "booking_url": "https://booking.laserairlines.com/flightresults/"
  }
]
```
