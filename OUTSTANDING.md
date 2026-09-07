# arc_temp_humid - outstanding work

Updated 6 September 2026. The repository rename and the main-site move are
complete and are no longer listed here; see `CHANGELOG.md` for what was done.

Everything described below is **not yet done**. What *is* done is in
`CHANGELOG.md`; the architecture is in `CLAUDE.md`.

---

## 1. UK overheating threshold - RESOLVED, no band applies

Both UK datasets keep `"threshold": None`. That is now a decision rather than a
placeholder.

CIBSE TM59 (2017) section 4.2 requires **both** of:

- **(a) Living rooms, kitchens and bedrooms.** Hours where dT is at least 1 K,
  May to September, no more than 3% of occupied hours. dT is measured against
  the **adaptive** limit (TM52 Criterion 1), not a fixed temperature.
- **(b) Bedrooms only.** Operative temperature between 22:00 and 07:00 not above
  **26 C** for more than 1% of annual hours - TM59 counts that as 32 hours, so
  33 or more is a fail.

The only fixed figure in TM59 is that 26 C, and it applies to bedrooms at night.
Both UK sensors are in **living rooms** (Grove `169502D1`, and Holywell
`0E3C12EC`, confirmed and renamed from "Internal Ambient"), so criterion (b)
does not apply to either and there is no fixed band to draw. Criterion (a) is
the adaptive test, which the comfort chart already serves.

If a bedroom sensor is added later, the threshold is currently per-region, so a
per-logger threshold would be needed to draw 26 C for that sensor alone.

### Two caveats before quoting TM59 numbers from this dashboard

- TM59 is a **design and modelling** methodology: standard occupancy profiles,
  dynamic simulation, and **operative** temperature. This dashboard plots
  **measured air** temperature, already listed as a known limitation in
  `adaptivecomfort.md`. Numbers from here are indicative of TM59, not a TM59
  assessment.
- Criterion (a) is adaptive against **TM52 Category II**
  (`0.33 x Trm + 18.8 + 3`), which is not a band the dashboard currently offers.
  See section 2.

---

## 2. TM52 / EN 16798-1 bands - DECIDED, not adding them

No TM59 claims are intended, so the UK datasets stay on **ASHRAE 55 80%** and no
TM52 bands are added.

Worth recording why that is a safe default rather than merely a convenient one:
across the UK running-mean range the TM52 Cat II upper limit sits about 0.7 to
0.9 K **above** ASHRAE's, so ASHRAE is the more conservative of the two. Nothing
is being flattered by the choice.

| Upper limit at | Trm 10 C | Trm 15 C | Trm 20 C |
|---|---|---|---|
| ASHRAE 55 80% (in use) | 24.4 | 26.0 | 27.5 |
| TM52 / EN 16798-1 Cat II | 25.1 | 26.8 | 28.4 |
| TM52 / EN 16798-1 Cat III | 26.1 | 27.8 | 29.4 |

On the current data the choice changes nothing anyway: TM59 criterion (a)
exceedance is 0.00% at Grove under every model, and at Holywell 0.53% on ASHRAE
against 0.27% on Cat II, both far inside the 3% allowance.

**If that ever changes** and a TM59 result does need stating, the line has to be
the one TM59 names. Adding TM52 Cat II and Cat III is two entries in
`COMFORT_MODELS` plus labels. Note that EN 16798-1 designates **Category III for
existing buildings**, which these retrofits are, so Cat III would likely be the
right one rather than Cat II.

## 3. Open-Meteo coordinates and date ranges - DONE

All three feeds now publish a coordinate that is not the building, and start
from the building's own record rather than an arbitrary date.

| Feed | Committed coordinate | Distance from building | Feed starts |
|---|---|---|---|
| `tz` | `-7.07, 39.30` | 0.56 km | 2023-02-12 |
| `grove` | `52.057774, -2.704697` | 0.60 km | 2026-07-01 |
| `holywell` | `52.926643, -4.230377` | 0.65 km | 2026-07-01 |

**The UK values are Open-Meteo model grid cell centres.** Requesting any point
inside a cell returns that cell's series, so this is byte-identical data while
the published number is a fixed feature of the weather model rather than a
private address. The town-centre coordinates used before were already in the
same cells, so the weather data was correct all along - what changed is what
gets published.

**Tanzania is rounded to 2dp instead**, because Open-Meteo serves that region
from a coarser global model and echoes the requested point rather than snapping
to a grid. Rounding costs nothing there: even 1dp (3.9 km away) returns
byte-identical data. The previous value carried seven decimals, which is
centimetre-level precision on a children's facility in a public repository.

### The 30-day lead-in, and a bug it fixed

Each `start_date` is the building's first sensor reading minus 30 days. The
lead-in is not padding: the EN 16798-1 running mean is seeded from its own first
day and decays that seed by alpha = 0.8 daily, so without a run-up the opening
fortnight of comfort points are measured against a running mean still carrying
an arbitrary starting value. Thirty days reduces the seed's weight to about 0.1%.

This turned out to matter for Tanzania, not just the UK. House 5's first sensor
reading is **2023-03-14**, while the Open-Meteo feed began **2023-03-15** - a day
*after* the data it exists to contextualise. Every House 5 comfort point in the
first couple of weeks of the record was being measured against a running mean
with no history behind it. The feed now starts 2023-02-12.

If sensors are ever installed earlier than the current start dates, move
`start_date` back to 30 days before the new earliest reading.

## 4. ARC UK Omnisense fetch - DONE and tested

`SITES["uk"]` uses site number **58345** ("Simmonds.Mills Retrofits") with the
shared `OMNISENSE_USERNAME` / `OMNISENSE_PASSWORD` secrets, and a `--site uk`
step runs daily in `update-dashboard-data.yml` beside the Tanzanian one.

Exercised in CI on 6 September 2026 via `debug-omnisense.yml`, which now takes a
site input so either site can be tested without waiting for the nightly run or
committing anything:

```
Omnisense fetch [uk] - 2026-09-06 17:29 UTC
  Date range: 2026-07-31 -> 2026-09-06
  Login successful.
  Download ready: 180855 rows
  Wrote 10.3 MB -> data/omnisense_uk/omnisense_uk_20260906_1729.csv
```

So the shared credentials do reach the UK site, as expected from the export
containing a House 5 sensor alongside the Grove and Holywell ones.

To re-test at any point: Actions -> Debug Omnisense fetch -> Run workflow ->
site `uk`. It downloads and discards, committing nothing.

## 5. UK feeds in the staleness monitor - DONE

`check_staleness.py` now watches `omnisense_uk`, `openmeteo_grove` and
`openmeteo_holywell` alongside the Tanzanian sources, with a per-sensor
drill-down for the four UK Omnisense loggers exactly as Tanzania has. Existing
labels gained a region so the two Omnisense feeds are distinguishable. The
status page needed no change: it iterates whatever `data/status.json` contains.

The first run with the UK sources in place also confirmed the Tanzanian problem
described in section 6.

---

## 6. Mkuranga Omnisense has stopped reporting

Not a pipeline fault and not fixable from here. The fetch runs and succeeds -
510 897 rows on 7 September - but the data inside it stops.

Reading the export sensor by sensor shows **two separate events**, which is the
useful part:

### Event 1: everything stopped on 4 September

| Sensor | Description | Last reading (EAT) |
|---|---|---|
| B3CE8C7C | Performance stats | 2026-09-04 09:17 |
| 3276012B | House 5 | 2026-09-04 14:16 |
| 320E02D1 | Weather Station | 2026-09-04 14:16 |
| 327601CD | House 5 | 2026-09-04 14:17 |
| 32760371 | House 5 | 2026-09-04 14:17 |
| 327601CB | House 5 | 2026-09-04 14:17 |
| 32760208 | House 5 | 2026-09-04 14:18 |
| 3276028A | House 5 | 2026-09-04 14:19 |
| 32760048 | House 5 | 2026-09-04 14:19 |
| 3276003D | House 5 | 2026-09-04 14:19 |
| 32760164 | House 5 | 2026-09-04 14:20 |
| 32760205 | House 5 | 2026-09-04 14:20 |

Eleven sensors stopped inside **four minutes** of one another. Sensors do not
fail in unison, so this is the **gateway**: power, internet, or the base station
itself. The gateway's own "Performance stats" record stops at the same time,
which is consistent. Start there rather than with any individual sensor.

### Event 2: two sensors stopped a month earlier, on 3 August

| Sensor | Description | Last reading (EAT) |
|---|---|---|
| 30B40014 | Sun | 2026-08-03 12:33 |
| 195701C1 | CO2 sensor in House 5 Living Room | 2026-08-03 12:35 |

These two stopped within two minutes of each other but **thirty-two days before
the rest**, and they are the two that are not plain House 5 room sensors. A
separate cause - batteries, a shared repeater, or physical disturbance. Note
`30B40014` is what the status page calls "Omnisense: weather station", which is
why that card has been showing 35 days stale while the others showed 3.

Both are visible on the status page now, the second only because the first was
investigated.

---

## 7. UK Copernicus climate data - DONE

Long-Term Mode now works on the UK datasets, with the United Kingdom region's
own ERA5 history and five CMIP6 projections. `CLIMATE_REGIONS["uk"]` points at
`data/hist_proj_uk/` and the three UK datasets name it.

The first export could not be used - it was produced with the variable on
"Change (relative to 1850-1900)" and so carried anomalies rather than
temperatures. The replacement is on "Value" and was checked three ways before
being installed:

- **Scenario identity.** The five model sets match the Tanzanian files exactly
  (9, 22, 23, 22, 27 models), so the files are SSP1-1.9, SSP1-2.6, SSP2-4.5,
  SSP3-7.0 and SSP5-8.5, none duplicated or missing.
- **Physical ordering.** The 2100 ensemble mean rises monotonically with
  scenario severity: 9.92, 10.27, 11.28, 12.43, 13.50 C.
- **Continuity with observation.** ERA5 reads 10.33 C in 2022 against the
  projections' 9.98 C, a gap of 0.34 C - tighter than the Tanzanian pair's
  1.06 C, so history and projection join cleanly on the chart.

### On the region

Drawn for the whole **United Kingdom**, not a free-draw region around the
buildings as Tanzania's was. That is defensible for climate projections, whose
grids are far coarser than weather, and it settles the earlier question of one
region or two: one covers both Grove and Holywell. Worth being a deliberate
choice rather than an accident of the export - if separate regions for Hereford
and Criccieth are wanted, it is a second folder and a second `CLIMATE_REGIONS`
entry.

---

## 8. Known limitation, not a task

ASHRAE 55 Section 5.4.1(a) requires that no heating is in operation. Nothing at
any site records heating status, and nothing in a temperature and humidity trace
reliably separates a heated room from a merely warm one.

The dashboard states this rather than inferring it: a bolded line in the sidebar
applicability panel directly beneath the quoted criterion, and in the tooltips
explaining the method. The 10-33.5 C running-mean gate is enforced
automatically and greys out readings the standard does not cover.

That gate now has teeth it did not have before, because the UK Open-Meteo feeds
reach back to March 2023: once UK winter indoor data exists, cold-weather
readings will be greyed and excluded from the comfort percentage. But **a mild
day with the heating on will still pass every test the dashboard can apply**.

Closing this properly needs a heating-status input, not a cleverer algorithm.
See `adaptivecomfort.md` section 2 for the full reasoning, including why
calendar-month filtering was considered and rejected.
