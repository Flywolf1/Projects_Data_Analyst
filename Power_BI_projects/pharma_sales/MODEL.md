# OlphaMarket — data model

The workbook's `Task` sheet asks for the two tables to be combined and *"optimized using Power
Query"*. So they are: **every transformation lives in this `.pbip`**, in M, and the only inputs are
the delivered workbook plus three small reference files. There is no pre-processing script in the
refresh path — open the file, hit Refresh, and the whole chain re-runs from the Excel.

```
input\Task for analytic position 2026.xlsx        (delivered, never edited)
input\allergy_generation.csv                      (antihistamine generations, per SKU)
input\sku_resolution_rules.csv                    (every deliberate deviation from source)
input\DIM_DATE.csv                                (calendar 2018-01-01 .. 2023-12-31)
        │
        ▼
   0_Source      p_ProjectRoot → Src_Workbook → Src_Data, Src_SKU        not loaded
   1_Reference   REF_GENERATIONS (not loaded), AUX_SKU_RESOLUTIONS
   2_Staging     SKU_CLEAN                                               not loaded
   3_Model       DIM_PRODUCT  DIM_CORPORATION  DIM_DATE  FACT_SALES  AUX_DQ_CHECKS
```

The query groups are numbered so the Power Query navigator reads top to bottom in the order the
data actually flows.

## Star schema

```
DIM_DATE[DateKey]            1 ─ * ┐
DIM_PRODUCT[ProductKey]      1 ─ * ┼──>  FACT_SALES
DIM_CORPORATION[CorpKey]     1 ─ * ┘
```

| Table | Rows | Grain |
|---|---|---|
| `FACT_SALES` | 5,948 | one row per **SKU × month** |
| `DIM_PRODUCT` | 107 | one row per SKU |
| `DIM_CORPORATION` | 27 | one row per corporation |
| `DIM_DATE` | 2,191 | daily, 2018-01-01 → 2023-12-31, marked as the date table |
| `AUX_SKU_RESOLUTIONS` | 8 | the cleaning decisions, with reasons |
| `AUX_DQ_CHECKS` | 17 | data-quality evidence, recomputed on every refresh |

`DIM_CORPORATION` joins the **fact**, not `DIM_PRODUCT`, to keep a flat star with no ambiguous
filter paths. `DIM_PRODUCT[Corporation]` is a descriptive text column only — do **not** add a
second relationship on it.

## What the transformation does, and why

**1 — Source, untouched.** `Src_Data` (5,964 rows) and `Src_SKU` (115 rows) promote headers and set
types. Nothing is filtered here, so a reviewer can always see the delivered data.

**2 — `Drug status` normalised first.** The source value is truncated: `OTC (bezrecepšu`, with no
closing bracket. It becomes `OTC` / `RX` **before** any rule compares against it. Order matters —
the previous version of this file filtered on the un-normalised value and silently matched nothing.

**3 — Duplicate SKUs resolved by rule, not by hand.** 8 SKUs appear twice in the masterdata
(16 rows). `input\sku_resolution_rules.csv` names, for each one, the conflicting attribute and the
value that is kept:

| SKU(s) | Conflict | Kept | Why |
|---|---|---|---|
| Zyrtec ×4, Xyzal ×2 | Corporation | **Viatris** | Viatris manufactures; MagnaPharm is the CEE distributor, so sales belong to the manufacturer |
| Rennie Tablet 1MG N24 | Molecule | **Antacids, other combinations** | One product carries one classification; the alternative label is the less specific |
| Nolpaza Tablet 20MG N14 | Drug status | **RX** | The RX→OTC switch happened in 2024-25, after this 2018-2023 window |

115 rows → **107, unique on the join key**. The rules load into the model as
`AUX_SKU_RESOLUTIONS`, so the "what we changed and why" is readable inside the report.

**4 — Generation enrichment.** `input\allergy_generation.csv` maps each of the 53 allergy SKUs to
its molecule, generation, sedation, INN, chemical class and main use. It is keyed per SKU, which
also resolves the 5 rows whose `Molecule` is the placeholder *Antihistamines for systemic use*
(FENKAROL → Quifenadine, Aviamarin → Dimenhydrinate, Allergicum → not an antihistamine).
Non-antihistamine SKUs are **labelled**, never left null — a null renders as `(Blank)` in every
slicer and legend, and grouping drops null keys, which would hide half the market.

**5 — Facts collapsed before anything is derived.** 32 rows form 16 duplicated `(SKU, Period)`
pairs. They are summed to one row per SKU-month **first**: a price taken off half a row is wrong,
and the later join would have multiplied them. 5,964 → **5,948**, totals unchanged. `SourceRows`
records which rows were combined.

**6 — `Period` parsed without a locale dependency.** `"2018 January"` is split against an explicit
month list rather than handed to `Date.FromText`, so the model parses identically on any machine.
`Year` and `MonthName` are emitted as separate columns, as the task asks. `DIM_DATE` likewise
parses `dd.MM.yyyy` with an explicit `Format` and `Culture`.

**7 — `Doses = UnitsPacks × PackSize`**, the additive denominator behind price per dose.

## Guard steps

Three queries end in a guard that raises rather than returning wrong data:

| Query | Refuses to load when |
|---|---|
| `SKU_CLEAN` | the join key is not unique — an unresolved duplicate would fan the fact table out and inflate sales |
| `DIM_PRODUCT` | `SKU` is not unique |
| `FACT_SALES` | a fact SKU has no product, or the grain is not one row per `(DateKey, ProductKey)` |

These are not decoration. The previous version of this file *did* fan out, silently, and the
totals looked plausible. Both guards were tested by deliberately breaking a resolution rule: the
refresh fails with `SKU_CLEAN is not unique on [Product name] - add the missing rule to
input\sku_resolution_rules.csv`.

`AUX_DQ_CHECKS` adds 17 non-fatal checks recomputed on every refresh — row counts at each stage,
orphan checks, and source-vs-model totals — each with its expected value and an OK / REVIEW status.

## Report pages

| # | Page | What it answers |
|---|---|---|
| 1 | Market Overview | Both markets together — trend, split, seasonality, year table |
| 2 | **Allergy market** | Seasonality, producers, forms, cost per dose, share in EUR |
| 3 | **Stomach pain market** | The same five cuts for the other market |
| 4 | Competitive Landscape | Who competes and how the ranking moved |
| 5 | Generations and Pricing | Antihistamine generations and what a dose costs |
| 6 | Bikarfen Baseline | Baseline for the 2024 relaunch |

Pages 2 and 3 are a matched pair — same layout, same five questions — so the two markets can be
read side by side. Each carries a page-level filter on `DIM_PRODUCT[Market]`, so every visual on
it is scoped to that market without a slicer having to be set.

Seasonality is cut differently on each: the allergy page splits months by
`DIM_DATE[AllergySeason]` (April–August, the Baltic pollen months) because that is the driver
there; the stomach page uses plain calendar `DIM_DATE[Season]`.

## Branding

Both brand colours are used throughout: **#00ada3** (lighter) and **#33797a** (darker).
`DIM_CORPORATION[ProducerGroup]` / `DIM_PRODUCT[ProducerGroup]` split every producer into
`Olainfarm` / `Other producers`, and that column is the legend on the producer and price charts —
so Olainfarm is picked out in the lighter colour on both market pages by the data, not by hand.
The Olainfarm measures (`Olainfarm Sales EUR`, `Olainfarm Share %`, …) sit in their own display
folder.

The logo from `Olpha_brand/OLPHA_logo-1.svg` appears on all six pages. The SVG is only a wrapper
around a base64 PNG, so `scripts/build_report.py` unwraps and downscales it to
`OlphaMarket.Report/StaticResources/RegisteredResources/olpha_logo.png`.

Two Power BI quirks are worth knowing before editing the report by hand:

* Desktop **deletes a registered custom theme file when it saves**, leaving only the first couple
  of palette slots in effect. Every chart therefore carries explicit `dataPoint` colours. The one
  exception is the 27-series corporation trend on *Competitive Landscape*, where a single flat
  colour would be unreadable — it keeps Power BI's palette.
* A **card's title defaults to hidden** even when its text is set, so the KPI labels need an
  explicit `show: true`.

Regenerate with:

```bash
python scripts/build_report.py
```

It is idempotent — it strips the pages and header elements it owns before rebuilding, so it can be
re-run over its own output. Close Power BI Desktop first, or the next save there will overwrite it.

## Price is not stored

Price is a ratio and must be aggregated as `SUM(value) / SUM(units)`, never averaged across rows,
so no price column exists on the fact:

```dax
Price per pack = DIVIDE ( [Sales EUR], [Packs] )
Price per dose = DIVIDE ( [Sales EUR], [Doses] )
```

⚠️ **Price per dose is only comparable within solid oral forms.** Syrups, drops, solutions, creams
and aerosols carry `PackSize = 1`, so one "dose" there is a whole bottle. Filter any price visual
by `DIM_PRODUCT[SolidOral]` or `[FormCategory]` — `[Price per dose solid oral]` does this already.

## Moving the project

Every path is built from the **`p_ProjectRoot`** parameter. Move the folder, change that one value
in Home → Transform data → Manage parameters, refresh. Nothing else contains a path. (Verified by
relocating the project to a different drive and refreshing to identical totals.)

## Rebuilding and checking

```bash
python scripts/build_pbip_model.py      # regenerate model.bim from source
python scripts/build_report.py          # regenerate report.json: styling, logo, market pages
python scripts/validate_pbip_model.py   # structure + row-level diff vs output/powerbi/*.csv
```

`validate_pbip_model.py` also checks the report: that every column and measure the visuals bind to
exists in the model, and that the registered resources are on disk. Power BI does not fail loudly
on a bad binding — it draws a "can't find field" placeholder, easy to miss across six pages.

`model.bim` is currently the copy Power BI Desktop wrote when it last saved. Re-running the build
script replaces it with the generated one, which Desktop then greets with *"There are pending
changes in your queries that haven't been applied"* — that banner just means Desktop has not yet
applied a model it did not write itself. Press **Ctrl+S** once and it clears; the data is already
correct either way. Close Power BI Desktop before regenerating, or your next save will overwrite
the new file.

`scripts/prepare_powerbi_model.py` is **no longer part of the refresh path**. It is kept as an
independent second implementation of the same pipeline: `validate_pbip_model.py` replays the M
logic in pandas and diffs it row by row against that script's output, so two separately written
implementations have to agree on all 5,948 fact rows and all 107 product rows — a much stronger
statement than two grand totals matching.

## Expected figures

**107 SKUs · 27 corporations · 5,948 fact rows · 72 months (2018-01 → 2023-12) ·
123,049,563.66 EUR · 37,512,447 packs.**
