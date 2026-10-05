# Updating the CAPE forecasts page

Operational runbook for the monthly forecast release. For what the page is and
how the files fit together, see [README.md](README.md).

---

## The short version

```bash
# 1. Generate this month's figures (also rewrites manifest.json)
cd /home/chc-source/shrad/Scripts/Operational/CAPE/Python
python3 CAPE_crop_yield_forecast_visualization_harvestat.py            # or give months: 2026-07 2026-08

# 2. Edit the narrative if you want to call out this month's forecast
#    (cape_forecasts_page/cape-harvestat/config.json -- see "Writing the narrative")

# 3. Publish
cd ../cape_forecasts_page/cape-harvestat
git status                     # see everything that changed
git add -A
git commit -m "Forecasts based on climate conditions through <Month> <Year>"
git push
```

Use `git add -A`, not a list of named files. A routine month only touches
`manifest.json` and `config.json`, but if you have also changed the page itself
(`script.js`, `index.html`, `style.css`) those have to go in the same push.
Pushing a new `config.json` without the `script.js` that understands it leaves
the live page showing nothing but an error.

**Step 3 is the one that is easy to forget.** The figures go live on
`data.chc.ucsb.edu` the moment the script writes them, but the crop, season and
forecast month selectors will not offer the new month until `manifest.json` is
pushed to this repository. See [Why the manifest has to be committed](#why-the-manifest-has-to-be-committed).

---

## Step 1: generate the figures

```bash
cd /home/chc-source/shrad/Scripts/Operational/CAPE/Python
python3 CAPE_crop_yield_forecast_visualization_harvestat.py            # or give months: 2026-07 2026-08
```

To (re)generate earlier forecast months, pass them as `YYYY-MM` arguments,
e.g. `python3 CAPE_crop_yield_forecast_visualization_harvestat.py 2026-07 2026-08`.
The viewer csv is read once for all of them.

The harvest year of each country comes from its `forecast_end_month` in the viewer
data: forecasts made after that month belong to next year's harvest. If a region
has countries in two harvest years for the same forecast month (e.g. Sudan 2026
and Tanzania 2027 in September), both sets of graphics are written and the month
selector lists them separately.

With no arguments, the script forecasts for the *previous* calendar month, so running it in, say,
early June produces the "based on data through May" graphics.

It writes figures to:

```
/home/chc-data-out/experimental/CAPE/harvestat/viewer/figures/regional/
    <harvest year>/<region>/<season>/<forecast month>/<crop>/<model>/
```

At the end of the run it rescans that whole tree and rewrites `manifest.json`
to two places:

| Path | Purpose |
|---|---|
| `/home/chc-data-out/.../harvestat/viewer/figures/manifest.json` | Archive copy, next to the figures |
| `cape_forecasts_page/cape-harvestat/manifest.json` | **The one the page reads.** Must be committed. |

Because it is a full rescan rather than an append, every forecast month still on
disk stays selectable — not just the one just produced. Deleting old figures
from the tree removes them from the page at the next run.

You should see this near the end of the output:

```
Refreshing the cape-harvestat manifest
Wrote manifest with N entries to /home/chc-data-out/.../figures/manifest.json
Wrote manifest with N entries to /home/chc-source/.../cape-harvestat/manifest.json
Remember to commit and push the manifest.json copy ...
```

If you only want to refresh the manifest — for example you copied figures in by
hand, or the plotting run was interrupted — you can rebuild it without
re-plotting anything:

```bash
python3 /home/chc-source/shrad/Scripts/Operational/CAPE/Python/cape_forecast_manifest.py harvestat
```

This is safe to run any time. It only reads the figure tree and writes the two
manifest files.

## Step 2: write the narrative

Optional. Skip it and the page falls back to each region's default text, which
stays accurate for any selection.

All wording lives in `config.json`. Each region has a default `summary` and
`caveats`, and a `notes` list that overrides them for particular combinations:

```json
"notes": [
  {
    "crop": "Sorghum", "season": "Gu", "month": "July", "year": 2026,
    "summary": "After masking for historical skill, ... **below normal** ...",
    "extraCaveats": [
      "For Eastern Africa, forecasts are presented for Sorghum (Gu season) only for Somalia, based on climate conditions through July 2026."
    ]
  }
]
```

Rules:

- A note applies when **every field it specifies** matches the reader's current
  selection. Fields it leaves out are wildcards, so
  `{"crop": "Sorghum", "summary": "..."}` covers every Sorghum selection.
- When several notes match, the one specifying the **most** fields wins.
- `summary` replaces the region summary.
- `caveats` **replaces** the region caveat list; `extraCaveats` **appends** to it.
- `**double asterisks**` render as bold. No other markdown is supported.

Old notes are worth keeping. They stay attached to their forecast month, so a
reader who selects an earlier month still gets the text that was written for it.

Also update `lastUpdated` at the top of `config.json` if you want the "Released
on" date in the header to change. The "Forecast graphics last refreshed" date
beside it comes from the manifest and updates itself.

## Step 3: publish

```bash
cd /home/chc-source/shrad/Scripts/Operational/CAPE/cape_forecasts_page/cape-harvestat
git status                     # confirm what changed
git add -A
git commit -m "Forecasts based on climate conditions through <Month> <Year>"
git push
```

**Push the page code together with the config.** `config.json` and `script.js`
are a matched pair: the config describes the data, the script knows how to read
it. Shipping one without the other breaks the live page. `git add -A` is the
safe habit here — this repository holds nothing you would not want deployed.

The site is served from the `main` branch of
`github.com/Shrad-Shukla1/cape-harvestat`, with no build step, so the push is the
deploy. Give it a minute and hard-refresh (Ctrl+Shift+R / Cmd+Shift+R) if you
still see the old version.

---

## Editing the methods and references

The **Methods & References** tab holds one shared copy of the methodology,
reachable from every region tab and from the "Methods, data sources and
references" link at the foot of each region page. There is only one copy, so
edit it once and it changes everywhere.

References live in the `references` list in `config.json`:

```json
"references": [
  {
    "authors": "Lee, D. et al.",
    "title": "Contrasting performance of panel and time-series data models for subnational crop forecasting in Sub-Saharan Africa",
    "source": "Agricultural and Forest Meteorology 359, 110213",
    "year": 2024,
    "url": "https://doi.org/..."
  }
]
```

Only `title` is required. Fill in `url` and the title becomes a link; leave it
empty and the citation renders as plain text. Punctuation is added for you, so
do not put a trailing full stop on `title` or `source` — an author field written
as `Lee, D. et al.` keeps its own period without doubling it.

The modelling framework prose, the EO data set list and the predictor table
above the references are rendered by `getMethodsSection()` in `script.js`; the
predictor cards themselves come from `cape_predictors` in `config.json`.

---

## Previewing before you push

```bash
cd /home/chc-source/shrad/Scripts/Operational/CAPE/cape_forecasts_page/cape-harvestat
python3 -m http.server 8000
```

Open `http://localhost:8000`, then stop the server with Ctrl+C.

Opening `index.html` directly by double-clicking will **not** work — the page
fetches `config.json` and `manifest.json`, and browsers block those reads over
`file://`. You need the local server.

Worth checking in the preview: the new forecast month appears in the dropdown
and is selected by default, the maps load, and your narrative shows up on the
right combination.

---

## Choosing what the page opens on

By default each region opens on the **latest forecast month in the season**,
which is normally what you want. To pin it instead, set `default` on the region
in `config.json`:

```json
"default": { "crop": "Sorghum", "season": "Gu", "month": "May", "year": 2026 }
```

`crop` and `season` are usually worth setting. Add `month` (and optionally
`year`) only when you deliberately want to hold the page on an older forecast —
and remember to remove them later, or the page will sit on that month forever.

Anything the manifest cannot honour is ignored, so a stale default quietly
degrades to the latest forecast rather than breaking the page.

---

## Why the manifest has to be committed

`data.chc.ucsb.edu` does not send CORS headers. A browser will happily load an
`<img>` from there, which is why the maps work, but it refuses a JavaScript
`fetch()` of a file from another origin. So the page cannot read the manifest
from the data server — it has to be served from this repository, alongside
`index.html`.

That is the whole reason for the two copies. The archive copy next to the figures
is for reference; the copy in this repo is the one that matters.

---

## Troubleshooting

**The page shows only "Error loading configuration".**
That wording comes from an *older* `script.js` than the `config.json` beside it —
a partial push, where the config went up but the page code did not. Run
`git status`; if `script.js`, `index.html` or `style.css` show as modified,
commit and push them. The reverse pairing (new script, old config) shows
"Error loading configuration or forecast manifest" instead. Either way the fix
is the same: push both halves together.

**The new month does not appear in the dropdown.**
The manifest was not pushed, or the run did not reach the manifest step. Check
that `manifest.json` in this repo contains the month:
`grep -c '"monthName": "July"' manifest.json`. If it is missing, rerun
`cape_forecast_manifest.py`. If it is present, `git status` — it probably needs
committing. Note `manifest.json` had to be `git add`ed the first time; after that
`git add manifest.json` picks up changes normally.

**The page is blank, or shows "Error loading configuration or forecast manifest".**
Almost always a JSON syntax error from hand-editing `config.json` — a trailing
comma or an unescaped quote. Check it with:
`python3 -c "import json; json.load(open('config.json'))"`

**Maps show broken-image placeholders.**
The manifest points at a figure that is not on the web server. Take the URL from
the broken image and `curl -I` it. If it 404s, the figure did not make it to
`/home/chc-data-out/...`; rerun the plotting for that month.

**A season or crop you expected is missing from the dropdown.**
The manifest only indexes files matching the current script's naming convention
(`East_Africa_2026_Sorghum_yield_forecasts.png` and siblings). Some directories
in the figure tree also hold output from an older convention
(`July_yield_fcst.png`, `yield_forecasts_percentage_departure_to_...`), and those
are deliberately ignored — their captions and baselines would not match. A
directory holding *only* old-convention files produces no entry at all. Check
what a directory actually contains before assuming the scan is at fault.

**The page shows the wrong month by default.**
Check for a pinned `month` in that region's `default` block in `config.json`.

**Only GB forecasts show, not LR.**
Intentional. `"model": "GB"` at the top of `config.json` selects which model the
page displays; without it, GB and LR would appear as duplicate, indistinguishable
forecast months in the dropdown. Change that value to switch models.

**The plotting run died partway through.**
The manifest step runs at the very end, so an aborted run leaves the manifest
stale. Fix the underlying error, then either rerun the script or just rebuild the
manifest with `cape_forecast_manifest.py` to pick up whatever figures did get
written.

---

## Adding a region

Add an entry to `regions` in `config.json` with an `id` matching the directory
name in the figure tree, lowercase with hyphens (`east_africa` → `east-africa`).
The tab, the selectors and the figures all follow from the manifest. A region
with no manifest entries shows a "coming soon" panel, so you can add it before
the forecasts exist.
