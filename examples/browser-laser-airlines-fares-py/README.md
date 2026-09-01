# Reliable airline fare discovery (Python)

Discover available fare families for one public airline route and date with a managed Solari cloud browser. The implementation uses the Laser Airlines booking flow as a concrete example, but the browser-and-parser pattern applies to other carriers with JavaScript-heavy booking sites.

## The problem

Travelers often need to compare fares across fragmented, JavaScript-heavy booking sites. Conventional server-hosted scrapers can be unreliable, resource-heavy, and difficult to debug when booking flows change.

## Why Solari

Flight aggregators need prices that come from the carrier's public booking flow, not unverified fare sites or opaque third-party agencies that may show stale, incomplete, or misleading prices. A managed Solari cloud browser lets an aggregator compare current fare families at the source while isolating browser workload from its application servers. It supports stealth and CAPTCHA handling and can record sessions when a booking flow changes. This example launches exactly one bounded session and does not enable a residential proxy.

For the concrete Laser booking flow, Solari addresses several practical problems:

- **The fares are produced by a browser application.** The public booking widget must bootstrap its JavaScript application, populate airport suggestions, operate a custom calendar, submit a search, and navigate to a separate results page. A simple HTTP request to the entry page does not produce the rendered fare cards.
- **Browser sessions are expensive and fragile to host alongside an aggregator.** Running Chrome locally adds CPU and memory pressure, browser-version management, and cleanup risks to the application server. Solari runs that workload remotely and releases the session in a `finally` block after one route/date search.
- **Booking bootstrap failures can be mistaken for “no flights.”** The upstream booking system can reject or expire a session before results load. In ordinary language, Solari's [stealth mode](https://docs.getsolari.com/stealth) runs a browser that looks less obviously automated to sites with bot defenses; it does not mean bypassing a login or other access control. This example also launches with `captcha=True`, which makes Solari's [managed solver](https://docs.getsolari.com/captcha) available if the page presents a supported challenge such as reCAPTCHA, hCaptcha, or Cloudflare Turnstile. The validated Laser search did not establish that a CAPTCHA appeared or was solved, so CAPTCHA handling should be understood as enabled protection rather than a required step in every search. Explicit URL and HTTP-status checks then keep a rejected session separate from a legitimate no-availability result.
- **UI failures otherwise provide little evidence.** When `--recording` is enabled, the remote session can be replayed to see whether an airport option failed to appear, the calendar changed, or the results page never loaded. This is much more useful than a generic timeout in a server log.
- **Interaction and extraction can fail independently.** Solari drives the live UI and returns the final rendered HTML. The separate static parser then extracts fares without browser calls, so a booking-flow failure can be distinguished from a results-page selector change.

Solari does not guarantee inventory or bypass an airline's access rules. It provides a controlled browser environment, clearer failure boundaries, and optional debugging evidence for a low-frequency public fare search.

## Production insight

The broader production application successfully used this approach for CCS → PMV, retrieving Economy Light, Economy Basic, Economy Plus, and Business Class fares directly from the public booking flow. The key value is not generic scraping; it is dependable price discovery for a real underserved travel market.

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

The program exits with a concise error when the API key is missing, the date is past or unavailable, the carrier reports no flights, Solari or the upstream site blocks the session, or the expected results-page selectors no longer match. It never prints the API key.

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
