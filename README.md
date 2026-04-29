# Jet Excellence Dispatch Calculator

A single-file public-facing web app for Citation X dispatch planning.

**Live:** [`dispatch-calculator-je.netlify.app`](https://dispatch-calculator-je.netlify.app/)

> **Public-facing build.** No pilot names, no per-pilot burn rates, no
> customer names. Burn-rate selection uses anonymous scenario options
> (Efficient / Average / Conservative).

## What's in the repo

| File         | Purpose |
| ------------ | ------- |
| `index.html` | The full app — HTML + CSS + JS + baked airport / fleet data, single self-contained file (~129 KB). |
| `README.md`  | This file. |

The deployed app has **no external runtime dependencies** other than two
`fetch()` calls to Open-Meteo for live OAT and winds-aloft. Everything else
ships inlined; the app works from `file://` once loaded.

## Tabs (6 total)

1. **Max Payload** — two side-by-side calculators. Generic baseline
   (anchored to fleet's heaviest BEW) and tail-specific (BEW auto-populates
   per fleet table).

2. **Nonstop Feasibility** — given DEP/ARR airports, ETD, and payload,
   returns trip-fuel build-up, range check, MZFW/MTOW/MLW check, and a
   plain-English verdict. Distance and headwind **auto-populate** from
   the airport pair + ETD: great-circle distance is computed from the
   inlined airport DB; cruise winds are pulled live from Open-Meteo
   when ETD is within the 7-day forecast window.

3. **Runway & OAT** — given tail, airport, available runway, planned
   takeoff weight (or MTOW), and OAT, returns weight-adjusted balanced
   field length plus the **critical OAT** — the exact temperature where
   BFL equals the available runway. Live OAT button hits Open-Meteo.

4. **Pre-Flight Gate** — single GO / NO-GO panel running 6 GOM 2.4 hard
   floors at once: range, fuel capacity (with wind correction), MZFW,
   MTOW, MLW, and runway BFL at planned weight + DEP OAT. Outputs
   ✓ GO / ⚡ GO with caution / ✗ NO-GO with per-check rationale.

5. **Tail Finder** — given route + payload + fuel, ranks fleet tails by
   which fits best (passes range, MZFW, MTOW, fuel-capacity). Failure
   reasons are surfaced for tails that don't fit.

6. **Fleet Activity** — historical reference: top routes (DEP→ARR pairs)
   and top airports flown. Aggregate counts only — no per-pilot data
   in this build.

## Cruise altitude selector

Most tabs that depend on cruise winds or fuel burn expose an FL selector
spanning **FL250–FL450** (default **FL450**). The selector remaps the
Open-Meteo pressure-level fetch (FL250 → 400 hPa, FL370 → 200 hPa,
FL450 → 150 hPa, etc.) and applies a Citation X SFC correction relative
to FL370 baseline:

| FL | hPa | SFC mult |
| -- | --- | -------- |
| 250 | 400 | 1.45 |
| 290 | 300 | 1.20 |
| 330 | 250 | 1.05 |
| 370 | 200 | 1.00 |
| 410 | 175 | 0.95 |
| 450 | 150 | 0.90 |

FL450 is the default because Citation X step-climbs above the jet stream
core after initial fuel burn, and burn drops at higher FL. FL250 is
included for MEL scenarios (e.g. cabin-pressure restrictions).

## Cross-tab features

### 📋 Copy for Slack

Every result panel has a Slack-share button. Click → a markdown-formatted
summary is copied to the clipboard, ready to paste into a Slack channel
or DM. Each tab has its own formatter capturing the relevant inputs +
verdict.

### 🖨 Print Trip-Pack

The print button (top-right of the tab nav) triggers `window.print()`
with a custom print stylesheet that strips the chrome and renders ONLY
the active tab's result panel as a clean one-page brief. Save-as-PDF
from the browser's print dialog to hand to crew.

### Save defaults

Selecting a tail or burn-rate scenario anywhere persists it via
`localStorage` so your preferred default loads next visit.

## Live-data fetches

| Endpoint | Used for | Refresh |
| -------- | -------- | ------- |
| `api.open-meteo.com/v1/forecast?hourly=temperature_2m` | Runway/OAT live OAT lookup | per-airport, per-ETD |
| `api.open-meteo.com/v1/forecast?hourly=wind_speed_<hPa>hPa,wind_direction_<hPa>hPa` | Nonstop wind correction at selected FL | per-route, sampled at 3 great-circle points |

Open-Meteo is free, requires no auth, and is CORS-friendly.

## Performance / weight math

- **Cruise speed:** Mach .88 ≈ 510 KTAS
- **Default cruise altitude:** FL450 (150 hPa) — selectable FL250–FL450
- **Jet-A density:** 6.7 lbs / gal
- **Reserve:** 45-min IFR final per GOM 2.4.3.b
- **Trip fuel (zero-wind):** `(distance / 510) × burn × sfc_mult × 6.7`
- **Trip fuel (wind-corrected):** `(distance / (510 − headwind_kt)) × burn × sfc_mult × 6.7`
- **Plus:** 100 nm alternate + 250 lbs taxi + 800 lbs climb
- **BFL interpolation:** linear in density altitude between
  `(SL, ISA)` → `bfl_sl_isa_ft` and `(5,000 ft, +20 °C)` → `bfl_5000_p20c_ft`
- **Density altitude:** `DA = elev + 120 × (OAT_C − ISA_at_elev_C)` where
  `ISA_at_elev = 15 − 1.98 × (elev / 1000)`
- **Weight scaling:** `BFL_actual = BFL_base × (W / MTOW)^1.5` (jet rule of thumb)
- **GOM hard floor:** any runway < 4,500 ft auto-flagged per GOM 2.4.3.d

## Privacy

This is the public-facing build:

- ✅ No pilot names anywhere in the bundle
- ✅ No per-pilot burn rates, hours, or per-leg history
- ✅ No customer names baked in
- ✅ Burn-rate selection is anonymous: Efficient (290 gal/hr) / Average
  (316 gal/hr) / Conservative (335 gal/hr)

## Deployment

This repo is wired to Netlify continuous deploy. **Pushing to `main`
auto-deploys** to `dispatch-calculator-je.netlify.app` within ~30s.

For manual / one-off deploys:

```powershell
# Netlify CLI (one-time setup)
npm install -g netlify-cli
netlify login

# Deploy current folder to production
netlify deploy --prod --dir .
```

## Local testing

Open `index.html` directly in any modern browser — `file://` works fine
because Open-Meteo allows `Access-Control-Allow-Origin: *`.

```powershell
start index.html
```

## Changelog

- **v5 (2026-04-28)** — Renamed product to **Dispatch Calculator**.
- **v4 (2026-04-28)** — Quote-Cost tab removed. Nonstop tab: distance
  and headwind auto-populate from DEP/ARR/ETD; no manual distance entry.
- **v3 (2026-04-28)** — Pre-Flight Gate, Tail Finder, Fleet Activity
  tabs. Live wind correction via Open-Meteo. Slack-share buttons. Print
  Trip-Pack stylesheet. localStorage saves defaults. Per-pilot data
  stripped from public bundle.
- **v2 (2026-04-27)** — Live OAT lookup. Airport DB expanded to ~265
  entries.
- **v1 (2026-04-26)** — Initial multi-tab expansion: Max Payload,
  Nonstop Feasibility, Runway & OAT.
- **v0** — Original Max Payload calculator (single tab).
