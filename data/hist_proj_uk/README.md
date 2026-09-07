# ARC UK climate data (incomplete)

`t-ERA5_timeseries_historic.csv` is in place and correct: absolute annual mean
temperature for the United Kingdom region, 1940-2025, around 8-10 C.

**The five CMIP6 projection files are still needed**, and the exports supplied on
7 September 2026 could not be used. They were produced with the variable set to
**"Change (relative to 1850-1900)"**, so they hold anomalies of roughly -1.4 to
+3.9 K rather than absolute temperatures. Plotted against the ERA5 line they
would sit far below it and appear to show the UK cooling.

For comparison, at 2100 under SSP2-4.5:

| | value |
|---|---|
| Tanzania (absolute, as used) | 28.24 C |
| UK export supplied (anomaly) | 1.82 |

Re-export the five scenarios with the variable on **"Climatology"** - the same
setting the Tanzanian files in `data/hist_proj/` used - and save them here as:

```
t-CMIP6_timeseries_SSP1-1.9.csv
t-CMIP6_timeseries_SSP1-2.6.csv
t-CMIP6_timeseries_SSP2-4.5.csv
t-CMIP6_timeseries_SSP3-7.0.csv
t-CMIP6_timeseries_SSP5-8.5.csv
```

The filenames matter: `build.py` globs `t-CMIP6_timeseries_SSP*.csv` and takes
the scenario name from the filename.

Then add a `CLIMATE_REGIONS["uk"]` entry in `build.py` pointing at this folder
and set `"climate_region": "uk"` on the three UK datasets. Until that happens
the UK datasets have no Long-Term Mode, which is deliberate - showing Tanzania's
projections under a UK building would be worse than showing none.

## Note on the region

These were drawn for the whole **United Kingdom**, not a free-draw region around
the buildings as the Tanzanian ones were. That is a reasonable simplification
for climate projections, whose grids are far coarser than weather - and it means
one region serves both Grove Cottage and Holywell Barn despite the 150 km
between them. Worth being a deliberate choice rather than an accident.
