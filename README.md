# CAPE crop yield forecasts page (HarvestStat version)

Parallel copy of the [FDW version](https://shrad-shukla1.github.io/cape-fdw/)
(`cape_forecasts_page/cape-fdw`), showing forecasts from the CAPE models trained on
HarvestStat Africa data (`/home/chc-data-out/experimental/CAPE/harvestat/viewer/`).
The page code is the same apart from the version label and cross-link in the
header, which are set by `version` in `config.json`.

Static page showing CHC/CAPE sub-national crop yield forecasts for East, Southern
and West Africa. Visitors pick a **region** (tab), then a **crop**, **season** and
**forecast month** from the selectors. A **Methods & References** tab carries a
single shared copy of the methodology, linked from every region page.

> **Updating the page each month?** See **[UPDATING.md](UPDATING.md)**.

## Files

| File | Maintained by | Purpose |
|---|---|---|
| `manifest.json` | **generated** | One entry per region / crop / season / forecast month that has graphics on disk. Drives the selectors. |
| `config.json` | by hand | Contact details, region narrative (summary and caveats), season labels, model choice, predictor table, references. |
| `index.html`, `script.js`, `style.css` | by hand | The page itself. |
| `UPDATING.md` | by hand | Monthly release runbook. |

Figures themselves are not in this repository. They are served from
`https://data.chc.ucsb.edu/experimental/CAPE/harvestat/viewer/figures/regional`, and
`manifest.json` holds the paths relative to that base.

## How it fits together

```
CAPE_crop_yield_forecast_visualization_harvestat.py
    │
    ├── writes figures ──> /home/chc-data-out/.../figures/regional/
    │                          <year>/<region>/<season>/<month>/<crop>/<model>/
    │                                      │
    │                                      └── served at data.chc.ucsb.edu ──┐
    │                                                                        │
    └── rescans that tree, writes manifest.json ──> this repo ──> script.js ─┴──> page
                                                                      ↑
                                                    config.json ──────┘
                                                    (narrative, labels)
```

`script.js` reads both files at load: `manifest.json` decides what can be
selected, `config.json` decides what is written about it. Nothing about the
available crops, seasons or months is hard coded in the page.

## Source

The generating scripts live outside this repository, in the CAPE operational tree:

- `Python/CAPE_crop_yield_forecast_visualization_harvestat.py` — produces the figures and
  refreshes the manifest at the end of each run.
- `Python/cape_forecast_manifest.py` — the manifest scanner. Also runnable on its
  own to rebuild `manifest.json` without re-plotting.
